---
layout: post
title:  "Alfresco logs and linux multitail"
date:   2015-04-28 23:56:45
comments: true
categories:
- blog
permalink: multitail-logs
description: couple of tips around logs in Alfresco.
---

Today I am going to introduce a small tool I've installed recently on my linux machine. The tool is called
`multitail`, it is a command line tool, and basically
it allows to watch files the same way `tail` does
except that in addition you can add styles and coloration. In fact it's mostly used to watch logs. I'll also give
easy tips to improve logs in Alfresco.

I configured the tool with Alfresco logs and I came with this result:

- alfresco.log
![alfresco.log]({{ site.url }}/assets/posts/multitail/repo_logs.png)

- share.log
![share.log]({{ site.url }}/assets/posts/multitail/share_logs.png)

- multi-files
![multi-view]({{ site.url }}/assets/posts/multitail/multi_logs.png)

> Of course it's not aimed to replace a complex monitoring tool in production. Nevertheless it's easy to play with and
 quite fun.

## Prefix logs

I don't know if you noticed on the screens but some lines are prefixed with `REPO_`, `SHARE` or `SOLR_`. Why doing
this? just to easily visualize from which application the logs come from and to do this you only have to chane the log
appenders
. I
find it
useful,
especially with
`catalina.log` where the logs are merged.
Later with multitail we will see together how to add style in order to highlight these keywords.

To add the `REPO_` prefix you need to modify the custom-log4j.properties file located in the
shared/classes/alfresco/extension
folder in tomcat.
{% highlight java%}
log4j.appender.File.layout.ConversionPattern=REPO_ %d{ISO8601} %x %-5p [%c{3}] [%t] %m%n
{% endhighlight %}

Then I've done the same for Share tough as far as I know there is no equivalent custom-log4j.properties with Share.
Instead I changed the normal log4j.properties in the WEB-INF folder. (I did it for Solr)

## Add the user logged in the NDC

NDC is quite a powerful feature that allows to create a context for your logs and add pieces information into the
logs for this particular context. It might not be really clear said like this, but it doesn't matter, what I want to
show here it's how to
 display the user logged in at the time logs are generated. If you glance at my second screenshot
 you can see the `User:admin` appeared in the logs. It seems the operation has been done by the admin.

To do this I added a request filter and push the user in the NDC. (for those who don't know about NDC there is many
information on the internet)

In your log properties
{% highlight java%}
// use log4j NDC to replace %x with User:username
log4j.appender.Console.layout.ConversionPattern=%d{ISO8601} %x %-5p [%c{3}] [%t] %m%n
{% endhighlight %}

## Start with Multitail

To install multitail on your machine run this command (according to your env it might be something else)
{% highlight sh%}
# CentOs, Fedora
$ yum install multitail

# Debian
$ sudo apt-get install multitail
{% endhighlight %}

There are many options coming with the tool, if you do a `man multitail` you can see them. I'm not going to explain all
of them cause I probably don't use myself half of them. Instead I prefer to show you examples:
{% highlight sh%}
# out of the box if you have local logs you can do (after Ctrl+C to exit the window)
$ multitail /home/pathtomylogs/share.log
# with two files
$ multitail /home/pathtomylogs/share.log /home/pathtomylogs/alfresco.log
{% endhighlight %}
This is not really exciting because it does the same as a cat for instance. What could be interesting is to see
remote logs.
{% highlight sh%}
# let's say devalf is a remote machine (the hostname is defined in my .ssh/config)
multitail -l 'ssh devalf "tail -200f /home/alfresco/logs/catalina.log"' -l 'ssh devalf
"tail -200f /home/alfresco/logs/share.log"'
{% endhighlight %}
You can of course increase the number of lines to output. If you press `h` you can see the help. I use a lot `b` when
 multiple files to select one and open it in a separated window, from this window scrolling is enable. So far there are
  no colors. To add them you'll need edit the .multitailrc

## Add colors -> .multitailrc

`.multitailrc` is the config file for multitail - by editing it you can define the syntax coloration for your log. It
allows to create several color scheme for the different kinds of logs you have. I will show you the color scheme I use
for Alfresco. The file should be located in your home directory, if the file doesn't exist you have to create it(~/
.multitailrc). Then as an example you can add these lines which are my color scheme for Alfresco logs.

{% highlight sh%}
colorscheme:alf_log
cs_re:white,,bold:\[[a-zA-Z_]*?\.[a-zA-Z_]*?\.[a-zA-Z_]*?]
cs_re_s:white,cyan:(User:[a-zA-Z]*)[^\s]
cs_re:blue:[0-9]{4}-[0-9]{2}-[0-9]{2}\s[0-9]{2}:[0-9]{2}:[0-9]{2},[0-9]{3}
cs_re:blue,,bold:REPO_
cs_re:magenta,,bold:SOLR_
cs_re:cyan,,bold:SHARE
cs_re:green,,bold:INFO
cs_re:yellow,,bold:WARN
cs_re:magenta,,bold:DEBUG
cs_re:red,,bold:ERROR
cs_re:red,,bold:FATAL
cs_re:green:^.*INFO.*$
cs_re:yellow:^.*WARN.*$
cs_re:magenta:^.*DEBUG.*$
cs_re_s:white,red:ERROR (.*)
cs_re:red:.*
{% endhighlight %}

In the first line is the name of he color scheme if you want to add a scheme then skip a line at the end and start a
new `colorscheme`. I called my color scheme `alf_log`.
Then
 all
other lines are regex associated with colors. You can notice each lines start either with `cs_re:` or `cs_re_s:`.
`cs_re:` means you want to apply the style for the text that matches the regex. The second will apply the style only on
the subpart which is between brackets (Eg. `cs_re_s:white,red:ERROR (.*)`).

The general syntax is the following:

> cs_re:FG_COLOR[,BG_COLOR[,ATTRIBUTE[/ANOTHER_ATTRIBUTE]]]:REGEX
> cs_re_s:FG_COLOR[,BG_COLOR[,ATTRIBUTE[/ANOTHER_ATTRIBUTE]]]:REGEX
>
> Eg:
>
> > * cs_re:color:regexp -> change the color of the text
> > * cs_re:,color:regexp -> change the background color
> > * cs_re:color,,bold: regexp -> change the color of text and set it to bold

Number of colors are limited unfortunately, I don't remember exactly the list but we've got almost all of them in my
 example.

Another important thing to know, it's that the order is important. For example `cs_re:red:.*` will be overloaded by the
line just above, and so on.

To run multitail with a color scheme you need ot add the option `-cS`:
{% highlight sh%}
multitail -cS alf_log -l 'ssh devalf "tail -1000f /home/alfresco/logs/alfresco.log"'

## More complex multitail
multitail -cS alf_log -l 'ssh devalf "tail -1000f /home/alfresco/logs/alfresco.log"' -cS alf_log -l 'ssh devalf "tail
-1000f /home/alfresco/logs/share.log"' -cS alf_log -l 'ssh devalf "tail -1000f /home/alfresco/logs/catalina.log"' -cS
alf_log -l 'ssh devsolr "tail -1000f /home/alfresco/logs/solr.log"' -cS alf_log -l 'ssh devsolr "tail -1000f /home/alfresco/logs/catalina.log"'
{% endhighlight %}


