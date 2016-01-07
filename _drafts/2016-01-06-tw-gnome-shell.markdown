---
layout: post
title:  "Gnome shell: my first extension"
date:   2016-01-06 23:56:45
comments: true
permalink: gnome-shell-tw
toc: true
description: Display in the top panel data from a web script.
---

Lately I've been playing with __gnome-shell__, I wanted to create an extension and finally came up with a result. 
The experience was actually painful due to a lack of documentation.
In this post will present the simple extension I've got and provide explanations whenever I can. 
However, this should be taken with carefulness since I've start with few days ago. Nevertheless the 
extension is pretty simple and I think it can help someone starting by me.

The extension looks like this:  
![tw-example.png]({{ site.url }}/assets/posts/tw-gnome-shell/tw-example.png)
It simply displays in the top panel a data retrieved from a web site. For information the web site I used
 is Transfer Wise but feel free to display whatever you like.

This post is not a tutorial to start with gnome-shell, it would be a really tough work. There is actually not so many doucentation online.
I will just recommend you this link [http://mathematicalcoffee.blogspot.fr](http://mathematicalcoffee.blogspot.fr/2012/09/gnome-shell-extensions-getting-started.html).

## Before starting

There are few commands and apps you should know before starting. 
 
* Open the "gnome-shell command popup" (not sure how it is called): ALT + F2
* Restart gnome-shell: open the popup(ALT + F2) + write "r" + press enter. (the screen should blink)
* Open looking glass: open the popup + write "lg" + press enter.
* Quit looking glass: ESC
* Gnome-shell crashed: 

{% highlight sh %}
$ ps -e | gnome-shell 
# ps aux | gnome-shell (when multiple results) 
# and then kill the process.
{% endhighlight %}

* Install Tweaks tool (allows to see gnome extensions installed and enable/disable/remove them): 
![tweaks-tool.png]({{ site.url }}/assets/posts/tw-gnome-shell/tweaks-tool.png)

{% highlight sh %}
# On fedora
$ dnf install gnome-tweak-tool
{% endhighlight %}
* Install gnome-shell Logs [https://wiki.gnome.org/Apps/Logs](https://wiki.gnome.org/Apps/Logs)

## Create a new project

Creating a new extension is easy, it's by writing this command: 
{% highlight sh %}
$ gnome-shell-extension-tool --create-extension
{% endhighlight %}
The command will prompt for a name, a description and a uui. You will be able to modify
 these metadata later if you like to. 
For my extension I entered:

>
* name: Transfer Wise
* description: Get CHF transfer rate
* uui: transferwise@smasue.github.com

At the end a code editor will open showing the default code for an extension:

{% highlight javascript %}
const St = imports.gi.St;
const Main = imports.ui.main;
const Tweener = imports.ui.tweener;

let text, button;

function _hideHello() {
    Main.uiGroup.remove_actor(text);
    text = null;
}

function _showHello() {
    if (!text) {
        text = new St.Label({ style_class: 'helloworld-label', text: "Hello, world!" });
        Main.uiGroup.add_actor(text);
    }

    text.opacity = 255;

    let monitor = Main.layoutManager.primaryMonitor;

    text.set_position(monitor.x + Math.floor(monitor.width / 2 - text.width / 2),
                      monitor.y + Math.floor(monitor.height / 2 - text.height / 2));

    Tweener.addTween(text,
                     { opacity: 0,
                       time: 2,
                       transition: 'easeOutQuad',
                       onComplete: _hideHello });
}

function init() {
    button = new St.Bin({ style_class: 'panel-button',
                          reactive: true,
                          can_focus: true,
                          x_fill: true,
                          y_fill: false,
                          track_hover: true });
    let icon = new St.Icon({ icon_name: 'system-run-symbolic',
                             style_class: 'system-status-icon' });

    button.set_child(icon);
    button.connect('button-press-event', _showHello);
}

function enable() {
    Main.panel._rightBox.insert_child_at_index(button, 0);
}

function disable() {
    Main.panel._rightBox.remove_child(button);
}
{% endhighlight %}
There is a complete description of this code in the offical documentation [here](https://wiki.gnome.org/Projects/GnomeShell/Extensions/StepByStepTutorial#myFirstExtension).

You can save and close the editor and reopen it in your favorite editor. The extensions are located in this folder:
{% highlight sh %}
.local/share/gnome-shell/extensions/
{% endhighlight %}

You should see a folder named after the uuid you gave. Inside there are three files:

>
* extension.js
* metadata.js
* stylesheet.css

To install the extension, open the Tweaks Tool, then you should see it in the list. Switch it on. The extension
is a button with an icon in the top panel. When you click on it a popup appears with "Hello World".

## Modify the extension

Now we have a new project we can start modifying it. For instance if you change the label.

{% highlight javascript %}
text = new St.Label({ style_class: 'helloworld-label', text: "My first extension!" });
{% endhighlight %}

Then restart gnome-shell to make the change effective. `ALFT+ F2, write "r" and press enter`.

### Step 1 clean up

We will actually reuse the hello world extension because it has similarities with what we want to build
. Indeed, it adds a component in the top panel. Of course, instead of an icon we would like a text gotten
from a web service. Also, we don't really want any popup. So let's do some clean up.

{% highlight javascript %}
const St = imports.gi.St;
const Main = imports.ui.main;

let text, button;
function init() {
	button = new St.Bin({
		style_class: 'panel-button',
		reactive: true,
		can_focus: true,
		x_fill: true,
		y_fill: false,
		track_hover: true
	});

	text = new St.Label({text: "Text"});
	button.set_child(text);
}

function enable() {
	Main.panel._rightBox.insert_child_at_index(button, 0);
}

function disable() {
	Main.panel._rightBox.remove_child(button);
}
{% endhighlight %}

I removed the function showHello and hideHello and change the button to display a text instead of the icon.
 
### Step 2
Let's go a bit further, though, it's not that simple without clear documentation. So to helped in this task I 
went on the Gnome extension web site [https://extensions.gnome.org](https://extensions.gnome.org/) and
searched for an extension close to mine and took a look a the code. I used this one (Forex indicator)[https://extensions.gnome.org/extension/867/forex-indicator/]
which is actually much more complex compared I what I wanted to build. 

### Logger

Logging  





