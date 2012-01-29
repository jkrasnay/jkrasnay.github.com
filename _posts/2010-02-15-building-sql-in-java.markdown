---
layout: post
title: Building SQL in Java
---

Handling SQL within Java applications can be tricky. For one thing, Java does
not support multi-line string constants, so you can end up with code that
looks like this:

{% highlight java %}
String sql = "select *" +
             "from Employee" +
             "where name like 'Fred%'";
{% endhighlight %}

This code is not just ugly but also error-prone: did you notice the missing
space between `Employee` and `where`?

A further challenge when working with SQL in Java is that we often need to
build the SQL dynamically. Suppose we were generating a query based on data
the user had entered through a search page. We would want to build the `WHERE`
clause dynamically, based on which data the user had entered:

{% highlight java %}
List<String> params = new ArrayList<String>();
StringBuilder sqlBuilder = new StringBuilder()
    .append("select * ")
    .append("from Employee ")
    .append("where 1=1 ");

if (name != null) {
    sqlBuilder.append("and name like ? ");
    params.add(name + "%");
}

if (age != null) {
    sqlBuilder.append("and age = ? ");
    params.add(age);
}

String sql = sqlBuilder.toString();
{% endhighlight %}

Note that we've added a dummy predicate (`1=1`) so we don't have to always
decide whether to prepend `where` or `and` to subsequent predicates. This is
not always necessary--we often have a predicate that's always required, such
as `active = 'Y'`--but it's awkward.

To address these problems, I created a simple class called `SelectBuilder`.
`SelectBuilder` is used like this:

{% highlight java %}
List<String> params = new ArrayList<String>();
SelectBuilder sqlBuilder = new SelectBuilder()
    .column("*")
    .from("Employee");

if (name != null) {
    sqlBuilder.where("name like ?");
    params.add(name + "%");
}

if (age != null) {
    sqlBuilder.where("age = ?");
    params.add(age);
}
{% endhighlight %}

Here, we don't need to use the dummy predicate, as `SelectBuilder` takes care
of properly adding our `where` and `and` keywords.  We also didn't have to
worry about adding spaces between the different SQL fragments.

Like `StringBuilder`, `SelectBuilder` uses setter chaining so we can write
code that reads like the SQL statement itself:

{% highlight java %}
SelectBuilder sqlBuilder = new SelectBuilder()
    .column("e.id")
    .column("e.name")
    .column("d.name as deptName")
    .from("Employee e");
    .join("Department d on e.dept_id = d.id")
    .where("e.salary > 100000");
{% endhighlight %}

`SelectBuilder` doesn't care about the order in which its methods are called.
Consider a base class that represents a report that can be customized by
subclassing:

{% highlight java %}
public class BaseEmpReport {
    public String buildSelect() {

        SelectBuilder sqlBuilder = new SelectBuilder()
            .column("e.id")
            .column("e.name")
            .from("Employee e");
            .where("e.salary > 100000");
        
        modifySelect(sqlBuilder);

        return sqlBuilder.toString();
    }

    protected void modifySelect(SelectBuilder builder) {
    }
}
{% endhighlight %}

We can subclass this report to add a column representing the employee's
department name:

{% highlight java %}
public class DeptReport extends BaseEmpReport {
    protected void modifySelect(SelectBuilder builder) {

        builder
            .column("d.name as deptName")
            .join("Department d on e.dept_id = d.id")
            .where("d.name = 'Marketing'");

    }
}
{% endhighlight %}

After writing this class, I discovered the [Squiggle SQL Builder][1] library.
Squiggle's `SelectQuery` class is similar to `SelectBuilder`, but it manages
more of the SQL syntax with Java objects and methods. For example, with
`SelectQuery` you might write:

{% highlight java %}
    Table orders = new Table("orders_table");
    SelectQuery select = new SelectQuery(orders);
    select.addColumn(orders, "id");
    select.addColumn(orders, "total_price");
    select.addCriteria(new MatchCriteria(orders, "status", MatchCriteria.EQUALS, "processed"));
    select.addCriteria(new MatchCriteria(orders, "items", MatchCriteria.LESS, 5));
{% endhighlight %}

The equivalent in `SelectBuilder` would read like this:

{% highlight java %}
    SelectBuilder select = new SelectBuilder()
    .column("id")
    .column("total_price")
    .from("orders_table")
    .where("status = 'processed'")
    .where("items < 5");
{% endhighlight %}

I find the latter to more readable and more flexible than the Squiggle code.

The full code for `SelectBuilder` is as follows:

{% highlight java %}
package ca.krasnay.common.sql;

import java.util.ArrayList;
import java.util.List;

public class SelectBuilder {

    private List<String> columns = new ArrayList<String>();

    private List<String> tables = new ArrayList<String>();

    private List<String> joins = new ArrayList<String>();

    private List<String> leftJoins = new ArrayList<String>();

    private List<String> wheres = new ArrayList<String>();

    private List<String> orderBys = new ArrayList<String>();

    private List<String> groupBys = new ArrayList<String>();

    private List<String> havings = new ArrayList<String>();

    public SelectBuilder() {

    }

    public SelectBuilder(String table) {
        tables.add(table);
    }

    private void appendList(StringBuilder sql, List<String> list, String init,
String sep) {
        boolean first = true;
        for (String s : list) {
            if (first) {
                sql.append(init);
            } else {
                sql.append(sep);
            }
            sql.append(s);
            first = false;
        }
    }

    public SelectBuilder column(String name) {
        columns.add(name);
        return this;
    }

    public SelectBuilder column(String name, boolean groupBy) {
        columns.add(name);
        if (groupBy) {
            groupBys.add(name);
        }
        return this;
    }

    public SelectBuilder from(String table) {
        tables.add(table);
        return this;
    }

    public SelectBuilder groupBy(String expr) {
        groupBys.add(expr);
        return this;
    }

    public SelectBuilder having(String expr) {
        havings.add(expr);
        return this;
    }

    public SelectBuilder join(String join) {
        joins.add(join);
        return this;
    }

    public SelectBuilder leftJoin(String join) {
        leftJoins.add(join);
        return this;
    }

    public SelectBuilder orderBy(String name) {
        orderBys.add(name);
        return this;
    }

    @Override
    public String toString() {

        StringBuilder sql = new StringBuilder("select ");

        if (columns.size() == 0) {
            sql.append("*");
        } else {
            appendList(sql, columns, "", ", ");
        }

        appendList(sql, tables, " from ", ", ");
        appendList(sql, joins, " join ", " join ");
        appendList(sql, leftJoins, " left join ", " left join ");
        appendList(sql, wheres, " where ", " and ");
        appendList(sql, groupBys, " group by ", ", ");
        appendList(sql, havings, " having ", " and ");
        appendList(sql, orderBys, " order by ", ", ");

        return sql.toString();
    }

    public SelectBuilder where(String expr) {
        wheres.add(expr);
        return this;
    }
}

{% endhighlight %}

[1]: http://joe.truemesh.com/squiggle/
