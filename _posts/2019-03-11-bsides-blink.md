---
layout: post
title:  "BSidesSF CTF Blink Writeup"
categories: [CTF]
date:   2019-03-11 00:00:00
---

The [BSidesSF](https://bsidessf.net) CTF happened about a week ago! It was the first CTF I've tried to compete in and I had a lot of fun on the `▣` team. This is the first in a series of writeups on the challenges I participated in.

Blink was the first mobile challenge in the event and served as a good introduction. The goal of the mobile challenges is to find a string (the "flag") using clues hidden in an app. Often the flag is in the app binary itself, but sometimes the challenge may lead you elsewhere afterwards. Blink was a relatively easy "101" challenge, only worth 50 points (the most difficult challenges in the CTF were worth 600 or more).
 <!--more-->

### Analysis

The first step in any challenge to open the prompt. It contained the text `Get past the Jedi mind trick to find the flag you are looking for.` and a link to an [apk file](https://en.wikipedia.org/wiki/Android_application_package) (Android app). I booted up an Android Emulator and used a simple [adb](https://developer.android.com/studio/command-line/adb) command to install the app.

{% highlight bash %}
$ adb install blink.apk
Success
{% endhighlight %}

Opening the app presents us with: 

{% figure [caption:"The app as it first opens."] %}
![Blink MainActivity screenshot](/img/blink-ss-1.png){: .align-left}
{% endfigure %}
If you aren't familiar with Android, an activity is similar to a window in a traditional desktop app or a page on a website. The hint is essentially saying we need to navigate to a different page.

The first place we should check for activities is the `AndroidManifest.xml` file. All of the accessible activities will be listed there. I used [Apktool](https://ibotpeaches.github.io/Apktool/) to disassemble the app.

{% highlight bash %}
$ apktool d ./blink.apk
I: Using Apktool 2.3.4 on blink.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: /var/folders/8j/h8wpd_757p729cx_91j50g2m0000gn/T/1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: Baksmaling classes.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
{% endhighlight %}

Then I opened up the Android Manifest..

{% highlight xml %}
<?xml version="1.0" encoding="utf-8" standalone="no"?><manifest xmlns:android="http://schemas.android.com/apk/res/android" package="com.example.blink" platformBuildVersionCode="1" platformBuildVersionName="1.0">
    <application android:allowBackup="true" android:debuggable="true" android:icon="@mipmap/ic_launcher" android:label="@string/app_name" android:roundIcon="@mipmap/ic_launcher_round" android:supportsRtl="true" android:theme="@style/AppTheme">
        <activity android:label="@string/title_activity_r2d2" android:name="com.example.blink.r2d2" android:theme="@style/AppTheme.NoActionBar"/>
        <activity android:name="com.example.blink.MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
    </application>
</manifest>
{% endhighlight %}

Here we can see two activities: `com.example.blink.MainActivity` and `com.example.blink.r2d2`. We know that `MainActivity` is where we saw Obi Wan earlier because it has an intent filter that makes it the default activity. The next step is to launch the `r2d2` activity. Let's whip out adb again and see what we can do.

{% highlight bash %}
$ adb shell am start -n com.example.blink/.r2d2

Starting: Intent { cmp=com.example.blink/.r2d2 }
Security exception: Permission Denial: starting Intent { flg=0x10000000 cmp=com.example.blink/.r2d2 } from null (pid=21352, uid=2000) not exported from uid 10089
{% endhighlight %}

Well, that didn't work. Looks like we don't have permission to launch this activity. Let's try something else!

[![xkcd 149](https://imgs.xkcd.com/comics/sandwich.png){: .align-left}](https://xkcd.com/149/)

Because my emulator is running a build of Android without Google Play Services in it, we can easily assume the root user.

{% highlight bash %}
$ adb root
restarting adbd as root

$ adb shell am start -n com.example.blink/.r2d2
Starting: Intent { cmp=com.example.blink/.r2d2 }
{% endhighlight %}

And it worked!

![Blink r2d2 screenshot](/img/blink-ss-2.png){: .align-left}

The flag was `CTF{PUCKMAN}`. The end! Tons of credit to @itsC0rg1 for putting the puzzle together, and BSidesSF for hosting.
