---
layout: post
title: Composable UIs with Wicket
---

Pages in a [Wicket](http://wicket.apache.org/) application consist of a
hierarchy of Java objects, each of which is bound to an element in an
HTML template.  A Wicket application will typically contain an abstract
base page that implements the page "chrome" such as the title, logos,
and menus, with concrete pages using Wicket's markup inheritance to add
content within the chrome.

Here's an example base page:

{% highlight java %}
public abstract class BasePage extends WebPage {

    public BasePage(PageParameters params) {
        super(params);
        add(new Label("title", new AbstractReadOnlyModel<String>() {
            public String getObject() {
                return getTitle();
            }
        }));
    }

    protected abstract String getTitle();

}
{% endhighlight %}

{% highlight html %}
<!DOCTYPE html>
<html xmlns:wicket>
  <body>
    <h1 wicket:id="title">_</h1>
    <wicket:child/>
  </body>
</html>
{% endhighlight %}

Here's a login page that extends the base page:

{% highlight java %}
public class LoginPage extends BasePage {

    private String username;
    private String password;

    public LoginPage(PageParameters params) {
        super(params);
        Form form = new Form("form", new CompoundPropertyModel(this));
        add(form);

        form.add(new TextField("username"));
        form.add(new PasswordTextField("password"));
        form.add(new Button("submit") {
            public void onSubmit() {
                // do the login
            }
        });

    }
}
{% endhighlight %}

{% highlight html %}
<!DOCTYPE html>
<wicket:extend xmlns:wicket>
  <form wicket:id="form">
    <ul class="form">
      <li>
        <label>Username</label> <input wicket:id="username" type="text">
      </li>
      <li>
        <label>Password</label> <input wicket:id="password" type="password">
      </li>
    </ul>
    <button wicket:id="submit">Login</button>  
  </form>
</wicket:extend>
{% endhighlight %}

The problem with this approach, especially in large applications, is
that it becomes difficult to maintain consistent HTML across all pages.
Not only can this result in pages with inconsistent appearance, but it
also makes it difficult to enforce new HTML coding standards since one
has to potentially visit all pages to effect any change.

At [Effective Registration](http://effectiveregistration.com/) we've
tackled this using an approach I like to call a "composable UI", in
which most pages are created purely in Java, with HTML templates
relegated to lower-level components that are plugged together as needed.

To begin with, rather than relying on markup inheritance our base page
implements a `RepeatingView` that allows us to add sub-panels to the
page without implementing an HTML template for every concrete page.
this theme will recur in a few places, so we create an interface to
represent a "component to which sub-panels can be added":

{% highlight java %}
public interface PanelContainer {
    public void addPanel(Panel panel);
    public String newPanelId();
    public void removeAllPanels();
}
{% endhighlight %}

Our base page now looks like this:

{% highlight java %}
public abstract class BasePage extends WebPage implements PanelContainer {
    
    private RepeatingView panelRepeater;

    public BasePage(PageParameters params) {
        super(params);
        add(new Label("title", new AbstractReadOnlyModel<String>() {
            public String getObject() {
                return getTitle();
            }
        }));
        add(panelRepeater = new RepeatingView("panel"));
    }

    protected abstract String getTitle();

    public void addPanel(Panel panel) {
        panelRepeater.add(panel);
    }

    public String newPanelId() {
        return panelRepeater.newChildId();
    }

    public void removeAllPanels() {
        panelRepeater.removeAll();
    }
}
{% endhighlight %}

{% highlight html %}
<!DOCTYPE html>
<html xmlns:wicket>
  <body>
    <h1 wicket:id="title">_</h1>
    <div class="panel"></div>
  </body>
</html>
{% endhighlight %}

Now in order to replace our login page, we need to create a few panels.
Here's a form panel:

{% highlight java %}
public class FormPanel extends Panel implements PanelContainer {
    
    private RepeatingView panelRepeater;
    private RepeatingView buttonRepeater;

    public FormPanel(String id, IModel model) {
        super(id);
        add(new Form("form", model));
        add(panelRepeater = new RepeatingView("panel"));
        add(buttonRepeater = new RepeatingView("button"));
    }

    public void addButton(Button button) {
        buttonRepeater.add(button);
    }

    public String newButtonId() {
        return buttonRepeater.newChildId();
    }

    public void removeAllButtons() {
        buttonRepeater.removeAll();
    }

    public void addPanel(Panel panel) {
        panelRepeater.add(panel);
    }

    public String newPanelId() {
        return panelRepeater.newChildId();
    }

    public void removeAllPanels() {
        panelRepeater.removeAll();
    }
}
{% endhighlight %}

{% highlight html %}
<wicket:panel xmlns:wicket>
  <form wicket:id="form">
    <ul class="form">
      <li wicket:id="panel">
      </li>
    </ul>
    <span wicket:id="button">_</span>  
  </form>
</wicket:panel>
{% endhighlight %}

Similarly, we would have to create panels that encapsulated a
`TextField` and it's label, a `PasswordTextField` and its label, and a
`Button`. However, this won't be wasted work, since we'll be able to
re-use these panels throughout the application.

(Creating panels that encapsulate form fields in a flexible way presents
its own challenges. I'll talk more about that in a subsequent post.)

Our login page now looks like this:

{% highlight java %}
public class LoginPage extends BasePage {

    private String username;
    private String password;

    public LoginPage(PageParameters params) {
        super(params);
        FormPanel formPanel = new FormPanel(newPanelId(), new CompoundPropertyModel(this));
        addPanel(form);

        form.addPanel(new TextFieldPanel("username"));
        form.addPanel(new PasswordTextFieldPanel("password"));
        form.addButton(new ButtonPanel(newButtonId()) {
            public void onSubmit() {
                // do the login
            }
        });

    }
}
{% endhighlight %}

Note that our page code is no more complicated than before, yet we don't
have to create `LoginPage.html`. This is a relief, since it can be
difficult to remember things like "a form is always laid out using `<ul
class='form'>`". We'll be guaranteed that all forms look the same.
Further, if we decide to change the form layout to use tables rather
than `<ul>` elements we can just change `FormPanel.html` (and perhaps
some of our form component panels) and the change will apply throughout
the application. Finally, once we have built up a library of panels, our
developers will be more likely to look for an existing panel rather than
rolling their own HTML when implementing a new page, further increasing
consistency.
