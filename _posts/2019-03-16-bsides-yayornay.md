---
layout: post
title:  "BSidesSF CTF Yay Or Nay Writeup"
categories: [Android, reverse engineering, CTF]
date:   2019-03-19 00:00:00
---

This is the second in a series of writeups on challenges from the [BSidesSF CTF](https://bsidessf.net). You can see a writeup of the first challenge, Blink, [here](/articles/2019-03/bsides-blink)

`Yay Or Nay` was the second mobile challenge in the CTF, this time worth 200 points. Like last time, we start out with a prompt and an apk file. This time the prompt came in a little more handy.

```
Keep track of places you would love / hate to see, by dropping markers with a simple click. Try YayorNay v1.2 today!

:::: Updated README :::: v 1.0 - Added short press, Yay support - Fix stability issues

v 1.1 - Added long press, Nay support - Add labels

v 1.2 - Populate from DB - Save to DB

To-do - Fix stability issues - Bug fixes - Implement feature to view by day
```

First things first, let's launch the app.

{% highlight bash %}
$ adb install YayorNay.apk
Success
{% endhighlight %}

The app opens up with some instructions ond how to use it an a button to get started.

{% figure [caption:"The app as it first opens."] %}
![YayOrNay MainActivity screenshot](/img/yayornay-main.png)
{% endfigure %}

*clicks get started*

{% figure [caption:"Rats"] %}
![YayOrNay blocked screenshot](/img/yayornay-stuck.png)
{% endfigure %}

Well, looks like our root-enabled emulator image isn't going to work out here. Let's launch a Google Play Services enabled one! Unfortunately these images are a little more locked down and we won't be able to (easily) get root on them.

{% figure [caption:"Trying again on a fully googleified emulator"] %}
![YayOrNay map screenshot](/img/first-map.png)
{% endfigure %}

Ok, so we have a map of San Francisco with a bunch of markers. Let's zoom around and see if anything sticks out. This may have been the worst part of the challenge, zooming and panning with an emulator can get tedious 😄

{% figure [caption:"Aha!"] %}
![YayOrNay map screenshot](/img/grid.png)
{% endfigure %}

Right in the middle of everything there's a grid of some sort. It seems like the next step should be to isolate it. Because I know from the challenge prompt that these pins are being loaded from a database I'll go looking for the app's sqlite db.

1\. Find the package that contains our yayornay app.
{% highlight bash %}
$ adb shell pm list packages | grep yayornay
package:com.example.yayornay
{% endhighlight %}
2\. Switch to that package's user
{% highlight bash %}
$ adb shell
generic_x86:/ $ run-as com.example.yayornay
{% endhighlight %}
3\. Find the app's database
{% highlight bash %}
generic_x86:/data/data/com.example.yayornay $ ls
cache  code_cache  databases  files  shared_prefs
generic_x86:/data/data/com.example.yayornay $ cd databases
generic_x86:/data/data/com.example.yayornay/databases $ ls
Location.db  Location.db-journal
{% endhighlight %}
4\. List the tables in that database
{% highlight bash %}
generic_x86:/data/data/com.example.yayornay/databases $ sqlite3 Location.db
SQLite version 3.18.2 2017-07-21 07:56:09
Enter ".help" for usage hints.
sqlite> .tables
android_metadata  locations
{% endhighlight %}
5\. Inspect the table schema
{% highlight bash %}
sqlite> .schema locations
CREATE TABLE IF NOT EXISTS "locations" (
	`date`	TEXT,
	`latitude`	REAL,
	`longitude`	REAL,
	`color`	REAL
);
{% endhighlight %}

We can see that the database has a list of lat,long pairs each with a date and a color. My first guess is that these correspond to the pins we saw on the map. Let's dump the data and see what we get.

{% highlight sql %}
sqlite> SELECT * FROM locations LIMIT 5;
02/03/2019|37.7842927|-122.4053593|120.0
02/03/2019|37.7838412|-122.4041845|0.0
02/07/2019|37.7863323436302|-122.42828886956|120.0
02/07/2019|37.7851367932719|-122.402353584766|120.0
02/07/2019|37.782343920755|-122.404699847102|0.0
{% endhighlight %}

Looks like a list of dates, coordinates in and around San Francisco, and the hues for green (120) and red(0)! The next thing I did was go off of the prompt and try to 