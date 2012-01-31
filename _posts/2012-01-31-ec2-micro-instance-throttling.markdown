---
layout: post
title: EC2 Micro Instance Throttling
---

I'm a big fan of Amazon EC2 micro instances. If you have modest
requirements, or if you need to split functionality onto separate
servers for security purposes, micro instances can be a very economical
way to do it.

But be careful. While micro instances perform very well in short bursts,
if you consume excessive CPU for more than a few seconds, the EC2
infrastructure may [throttle back your
VM](http://gregsramblings.com/2011/02/07/amazon-ec2-micro-instance-cpu-steal/),
perhaps by more than a factor of thirty!

I stumbled across this when our monitoring tools were regularly
reporting trouble connecting to our micro instance servers each day
around 6:00 in the morning. It turns out that's when the system was
running [rkhunter](http://www.rootkit.nl/projects/rootkit_hunter.html),
which can run for a minute or more at high CPU loads looking for
malware. It seemed a shame to upgrade to a "small" instance, at four
times the cost, just to satisfy a non-critical, daily batch job, so I
set out to find some way to run rkhunter without running afoul of
Amazon's throttling mechanism.

The solution is as follows.

First I had to figure out the threshold at which Amazon would stop
throttling the VM. By playing with the script shown on
[gregsramblings.com](http://gregsramblings.com/2011/02/07/amazon-ec2-micro-instance-cpu-steal/)
I figured out that one second of execution and nine seconds of sleep
worked indefinitely without throttling.

To apply this to rkhunter, it helps to know a little about
signals, which are the Unix/Linux way to manage running processes. You
send a signal to a process using the `kill` utility, passing the name of
the signal to send and the pid of the target process. You can use
`pkill` to identify the process by name if you don't know the pid.

If you run `kill` without specifying a signal it assumes the `TERM`
signal, which terminates the process. Obviously that doesn't help much
here; however, there are two signals that do help. The `STOP` signal
pauses the target process, removing it from the operating system's
scheduling queue. The `CONT` signal resumes the process. We can use
these two signals to ensure rkhunter goes about its business in sprints
of one second, taking a nine second breather in between and avoiding the
wrath of the Amazon throttler.

OK, enough theory. To implement this I renamed
`/etc/cron.daily/rkhunter` to `/etc/cron.daily/rkhunter.norun`. The dot
in the filename prevents `cron` from running this script directly. Then
I created a replacement called `/etc/cron.daily/rkhunter-throttled` that
looks like this:

{% highlight bash %}
#!/bin/sh

/etc/cron.daily/rkhunter.norun &

while true; do
    sleep 1
    if ! pkill -STOP -x rkhunter >/dev/null 2>&1; then break; fi
    sleep 9
    if ! pkill -CONT -x rkhunter > /dev/null 2>&1; then break; fi
done
{% endhighlight %}

Note the `-x` argument, which asks `pkill` to match the process name
`rkhunter` exactly. If we don't do this the script pauses itself
(because it's called `rkhunter-throttled`) and hangs forever.

This approach should work for any long-running background process on
your instance, so long as you don't mind it taking ten times longer than
usual.
