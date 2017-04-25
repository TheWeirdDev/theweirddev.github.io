---
layout: post
title:  "معرفی زبان برنامه نویسی والا"
date:   2017-04-25 12:55:20
comments: true
permalink: introduce-vala
published: true
description: نگاهی میندازیم به زبان Vala
---

شاید اسم زبان Vala به گوشتون خورده باشه . در این پست قصد داریم تا کمی باهاش آشنا بشیم و ببینیم چه چیزی اون رو خاص و جذاب میکنه.

## مقدمه
ربان والا در سال 2006 به وسیله ی توسعه دهنده های گنوم ایجاد شد. هدف این بود که ساختن برنامه های گرافیکی Gtk نسبت به قبل آسون تر بشه و برنامه نویس ها بتونن از شی گرایی و بسیاری از قابلیت های جدید زبان های برنامه نویسی استفاده کنن.
یکی از چیز هایی که خیلی این زبان رو جالب میکنه کامپایلر اونه. برخلاف بقیه کامپایلر ها که کد ها رو به زبان ماشین ترجمه می کنن، کامپایلر والا کد هارو به زبان سی ترجمه میکنه و بعد به وسیله ی gcc اونها رو کامپایل میکنه !

والا بسیار شبیه به سی شارپ ساخته شده . اگر برنامه نویس سی شارپ یا جاوا هستید خیلی راحت میتونید با والا دوست بشید. 

> بعضی از قابلیت های مهم و مدرن والا رو میتونید در لیست زیر ببینید:
>
	* Interfaces
	* Properties
	* Signals
	* Foreach
	* Lambda expressions
	* Type inference for local variables
	* Generics
	* Non-null types
	* Assisted memory management
	* Exception handling

## اولین قدم در یادگیری یک زبان جدید: نوشتن یک برنامه ساده HelloWorld

دو نوع برنامه hello world رو در والا بررسی میکنیم. یکی تحت کنسول و یکی هم به صورت گرافیکی:

### نوع اول :
از hello world تحت خط فرمان شروع میکنیم 

برای شروع اول vala رو روی توریع تون نصب کنید. برای آرچ لینوکس از این طریق نصب میشه:

{% highlight bash %}
sudo pacman -S vala
{% endhighlight %}
برای توزیع های دیگه هم اگر سرچ کنید به راحتی روش نصبش رو پیدا میکنید.


{% highlight c# %}
static int main (string[] args) {
    stdout.printf("Hello World!\n");
    return 0;
}
{% endhighlight %}
برای کامپایل اولین برنامه، کد بالا رو داخل یه فایل به اسم `HelloWorld.vala` کپی پیست و ذخیره کنید. بعد به وسیله ی کامپایلر والا، `valac` کامپایل کنید
{% highlight bash %}
valac HelloWorld.vala
{% endhighlight %}

خب اگه کد هارو به درستی وارد کرده باشید میبینید که یه فایل به اسم HelloWorld ساخته شد . تبریک میگم ! اولین برنامه ی vala تون رو کامپایل کردید. برای اینکه اجراش کنید و نتیجه رو ببینید، میتونید از طریق خط فرمان اجراش کنید:
{% highlight bash %}
./HelloWorld
{% endhighlight %}
همونطور که میبینید `!Hello World` در خط فرمان نوشته میشه و برنامه به پایان میرسه.

### نوع دوم:
بخش خوب ماجرا از اینجا شروع مبشه. میتونید ببینید که کد های والا نسبت به سی چه قدر خوانا تر و تمیز تر هستن. این به برنامه نویس ها اجازه میده به جای نوشتن کد های زیاد و نا خوانا و شلوغ روی اصل برنامه تمرکز کنن و از خیلی از سختی های زبان سی رها بشن.
مثلا این hello world گرافیکی که با زبان والا نوشته شده رو در زیر می بینید:
{% highlight c# %}
using Gtk;

int main (string[] args) {
    Gtk.init (ref args);
    
    var window = new Window ();
    window.title = "Hello, World!";
    window.border_width = 10;
    window.window_position = WindowPosition.CENTER;
    window.set_default_size (350, 300);
    window.destroy.connect (Gtk.main_quit);
    
    var label = new Label ("Hello, World!");
    
    window.add (label);
    window.show_all ();
    
    Gtk.main ();
    return 0;
}
{% endhighlight %} 
 فایل رو به اسم HelloGtk.vala ذخیره کنید.

 کامپایل این کد کمی نسبت به قبلی فرق میکنه. چون از کتابخانه Gtk در برنامه استفاده کردیم باید به کامپایلر بگیم از پکیج gtk+ استفاده کنه:

{% highlight bash %}
valac --pkg gtk+-3.0 HelloGtk.vala
{% endhighlight %}

اجراش کنید و می بینید که برنامه به صورت گرافیکی اجرا میشه.به همین سادگی با چند خط کد تونستیم یک برنامه Gtk ایجاد کنیم.
![screenshot]({{ site.url }}/assets/images/vala-gtk.png)


### مقایسه با زبان سی 
 میدونید معادل همین برنامه در زبان سی چی بوده ؟
با دستور :
{% highlight bash %}
valac -C HelloGtk.vala
{% endhighlight %}


 کامپایلر فایلی به اسم HelloGtk.c تولید میکنه که میتونید کد های تبدیل شده از والا به سی رو داخلش ببنید:
{% highlight c %}

#include <glib.h>
#include <glib-object.h>
#include <stdlib.h>
#include <string.h>
#include <gtk/gtk.h>

#define _g_object_unref0(var) ((var == NULL) ? NULL : (var = (g_object_unref (var), NULL)))

gint _vala_main (gchar** args, int args_length1);
static void _gtk_main_quit_gtk_widget_destroy (GtkWidget* _sender, gpointer self);

static void _gtk_main_quit_gtk_widget_destroy (GtkWidget* _sender, gpointer self) {
	gtk_main_quit ();
}

gint _vala_main (gchar** args, int args_length1) {
	gint result = 0;
	GtkWindow* window = NULL;
	GtkWindow* _tmp0_ = NULL;
	GtkLabel* label = NULL;
	GtkLabel* _tmp1_ = NULL;
	gtk_init (&args_length1, &args);
	_tmp0_ = (GtkWindow*) gtk_window_new (GTK_WINDOW_TOPLEVEL);
	g_object_ref_sink (_tmp0_);
	window = _tmp0_;
	gtk_window_set_title (window, "Hello, World!");
	gtk_container_set_border_width ((GtkContainer*) window, (guint) 10);
	g_object_set (window, "window-position", GTK_WIN_POS_CENTER, NULL);
	gtk_window_set_default_size (window, 350, 300);
	g_signal_connect ((GtkWidget*) window, "destroy", (GCallback) _gtk_main_quit_gtk_widget_destroy, NULL);
	_tmp1_ = (GtkLabel*) gtk_label_new ("Hello, World!");
	g_object_ref_sink (_tmp1_);
	label = _tmp1_;
	gtk_container_add ((GtkContainer*) window, (GtkWidget*) label);
	gtk_widget_show_all ((GtkWidget*) window);
	gtk_main ();
	result = 0;
	_g_object_unref0 (label);
	_g_object_unref0 (window);
	return result;
}

int main (int argc, char ** argv) {
#if !GLIB_CHECK_VERSION (2,35,0)
	g_type_init ();
#endif
	return _vala_main (argv, argc);
}
{% endhighlight %}

همون طور که میبینید کد ها نسبت به والا بیشتر و شلوغ تر هستن و هرچی برنامه ها بزرگ تر و پیچیده تر میشن خوانایی کدها کم تر میشه.

### محیط های برنامه نویسی والا
استفاده از IDE ها برای برنامه نویسی کار رو بسیاز راحت تر و دلپذیر تر میکنه. برای والا هم IDE های خوبی مثل `Anjuta` و `Gnome Builder` وجود داره که با قابلیت های code completion و highlighting و بسیاری از موارد دیگه ، کار رو بسیار راحت تر میکنه.
 من Anjuta رو پیشنهاد میکنم چون قابلیت های بیشتری داره.


### نتیجه گیزی
 اگه به دنبال برنامه نویسی اپ های دسکتاپ در لینوکس هستید یا از امتحان کردن چیز های جدید خوشتون میاد حتما به والا یک فرصت بدید و مطمئن باشید که سربلند خواهد بود.

در صورتی که مشتاق هستید بیشتر بدونید و یاد بگیرید میتونید از لینک های زیر استفاده کنید:

[ویکی پدیا](https://en.wikipedia.org/wiki/Vala_(programming_language))

[ویکی گنوم](https://wiki.gnome.org/Projects/Vala/Documentation)

[مثال های سایت گنوم](https://wiki.gnome.org/Projects/Vala/GTKSample)

[آموزش های ویدئویی اقای تیموریان](http://www.aparat.com/aliireeza_t)

