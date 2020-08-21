---
layout: post
title:  "BSidesSF CTF Weather Companion Writeup"
categories: [CTF]
date:   2019-04-05 00:00:00
---

This is the third in a series of writeups on challenges from the [BSidesSF CTF](https://bsidessf.net/). You can see a writeup of the first challenge, Blink, [here](/articles/2019-03/bsides-blink) and the second, Yay or Nay, [here](/articles/2019-03/bsides-yayornay).

`Weather Companion` was the final mobile challenge in the CTF, this time worth 350 points! We’re provided with an apk file and a prompt that doesn’t set us up with much:

```
A simple weather application that fetches and displays the weather. What hides within?
```

 <!--more-->

Launching up the app we find that they were true to their word. A simple weather application:

{% figure [caption:"The app as it first opens."] %}
![Weather Companion MainActivity screenshot](/img/weather_main.png)
{% endfigure %}

Given that the only information we have is that we are fetching the weather, I launched Charles Proxy to try and get some more information on the network request. It turned out that the request was using TLS and we couldn’t see the payload. Having had to go through the pain of getting the various keystores on an Android device and Charles Proxy working together before, I put this strategy off for now. The one thing I did learn was that the weather request was going to a Google owned IP address.

Next I decompiled the apk with [jadx](https://github.com/skylot/jadx). The application was super obfuscated (presumably via [ProGuard](https://developer.android.com/studio/build/shrink-code)). All of the imported packages were renamed to single letters, but the main application was left easy to find.

{% figure [caption:"The obfuscated file tree"] %}
![Weather Companion decompiled files](/img/weathercompanion_file_tree.png)
{% endfigure %}

Going straight to the MainActivity we find an `onCreate()` method that instantiates an `example.MyApplication.a` object and calls `a#execute().`

{% highlight java %}
public class MainActivity extends c {
    public void onCreate(Bundle bundle) {
        super.onCreate(bundle);
        setContentView((int) R.layout.activity_main);
        c().a((Toolbar) findViewById(R.id.toolbar));
        try {
            new a(this, getApplicationContext()).execute(new Void[0]);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
{% endhighlight %}

The `a` class is pretty long, I’ve removed some parts for brevity:


{% highlight java %}
public final class a extends AsyncTask<Void, Void, String> {
    private String doInBackground() {
        String str2 = "weather-companion";
        String str3 = "weather.json";
        String str4 = null;
        try {
            Utils utils = new Utils(this.b);
            String a = Utils.a("LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUV2QUlCQURBTkJna3Foa2lHOXcwQkFRRUZBQVNDQktZd2dnU2lBZ0VBQW9JQkFRQ2JOYUpYN3FaMlNlYzQKNVc0aXIreVlYSjNJd2paOGZ3eXcwUFpTb2lUYi9pSnFUY0trL2x0alA2MVRySkhCNU1xS202VnovV0d3N0dTbQpuZDIxeE1GcWNsd3VHOE43Zit6aElLMFh1dlJCclMrY01FaHcwUmJITlViY08zWmFnaEtmZmFMVGh6UFk0eDJyCkhoL044bFVZaTRUQjhXQUdhQXpDSkgzcFVpOXJURzQrdWN4TzZwTU56My9FTnp5T1NtaWhSOVhiNU9rTVkvQXEKN05KN0JBd0g4b1NXUmxUbGxhQzhDaGw4d0RkdW42ZktXZ1lZbW1jQldYU2pyb3B1OGY2TVI3dk1NM0YwUEV1YQpZQnoxWnB4VnZZYzFpaU1TMFdpTnRBYTFaQnVXaFJlbkFpVGZ3T0N0NHJJU2ZiZ1lTcUNUU29LNXRPV1M1MnRNCg==", 0);
            String str5 = new String(utils.dks());
            String str6 = new String(utils.ss("CnoM+J76ft/1b1qCH4j4+/0h/EHG+iWbhzfijs5a7XHKINC8ach6YLbhAyUm9+mI\nGuGVAklGeeN5InzSubRcrL8BSobAIygkXw9aSioF2wj2TYMDYncA0KVLR0nkEAWf\nPFlp2YQLIIw0nowmk0PPSp0F+DXOtDP7i7xcFZWzVjy7ScWz/G9bnxqilMAmg3MH\n+QGEzCKu/JZ5p6w9iqmTXd3VyzUI/FFmIHsjDnkS8ImDyNv69seh/QFgG8u7PcTq\nYRm8F0dd7hoof7tBQX/MejFDui8qbrtMTh0agHEsnIY7NalXTWId2FYurIuZuWrM\nz94zTZR2BjXOtSKSFJpTETuZ/JkXTiNT0WJgTjMgawj+eNpEMSMNNcsSdVWSuKAr\n", 390));
            String a2 = Utils.a("M5VXSRDXNBVGCZDWOBUWCWJYG52FAK3BOIYU2VSYORSVGMLRINEW2YSHMZREO6KLJV3SWNLPKEXWY5KINRMHGOLWKFTUO53ILFGVCWAKNJCVGNTTJ5ITKNCJMREU24SPINEG4Q2WMZ5GWMDSONKHM22KPFFWQMKNGJDTANKDJFHUERRXJZUVSRZVKBSFEQSUGVAW6R2BIJCXOVAKIZHHEUTFIR5DSRDQIFYHGYLOINHGGV2ZHFIG6TKJMUZWYQZTKRJTIS3LG5WDG6KSJZHE44ZTONEGY5BXJZYVQ5DKPFKEKK2LHAZS6VIKJVQSWUCIMRYTAWKTJBBVCV2VGM2VE2DPOJCDOVCPHEZDERBTG5TUMRCZGA4DA6LDNBBEYTJZGZLGC42RKRGWKZSKMRCEQU2VJYYTO2IKLJZEURDBNR2WS6DQKA3S6YRQOFKGYUCCHFFDQ5KZNJEDE5DMIZLSWRSCIF2TS22DM5MUEY3IHBLHM6KCPJWEOYLZGVZDAN3KN5JGU5IKMNSG4VTGOJDUCVLELJJEY6CBGVUHESLBJN3EWRTZGI4EGYLBJNLC63C2JZETSTTNMRNG45DKGZQUSUJQOJ5FKV3JMRHVC43NNIZG6NAKG5LGGQLSNQ2VKTTVOJUHQNDVNBTTER2DNRRVKSKILE2VSZCFPBEGMRLVNJBHKNTVJVIWIWTKIJYFQOBXOVXWCZ3HIM3WU52KMFXFCQYKNRAXAVCQMMZTINZYM4XW22DPHFJTKZKEJV3T2PIKFUWS2LJNIVHEIICQKJEVMQKUIUQEWRKZFUWS2LJNBI======", 1);
            StringBuilder stringBuilder = new StringBuilder();
            stringBuilder.append(a);
            stringBuilder.append(str5);
            stringBuilder.append(str6);
            stringBuilder.append(a2);
            a = stringBuilder.toString();
            StringBuilder stringBuilder2 = new StringBuilder("{\"type\": \"");
            stringBuilder2.append(utils.a.getResources().getString(R.string.type));
            stringBuilder2.append("\",");
            str5 = stringBuilder2.toString();
            StringBuilder stringBuilder3 = new StringBuilder();
            stringBuilder3.append(str5);
            stringBuilder3.append("\"project_id\": \"");
            stringBuilder3.append(Utils.a(utils.b));
            stringBuilder3.append("\",");
            str5 = stringBuilder3.toString();
            stringBuilder3 = new StringBuilder();
            stringBuilder3.append(str5);
            stringBuilder3.append("\"private_key_id\": \"");
            stringBuilder3.append(Utils.a());
            stringBuilder3.append("\",");
            str5 = stringBuilder3.toString();
            stringBuilder3 = new StringBuilder();
            stringBuilder3.append(str5);
            stringBuilder3.append("\"private_key\": \"");
            stringBuilder3.append(a.replace("\n", "\\n"));
            stringBuilder3.append("\",");
            a = stringBuilder3.toString();
            stringBuilder3 = new StringBuilder();
            stringBuilder3.append(a);
            stringBuilder3.append("\"client_email\": \"");
            stringBuilder3.append("weather-companion-service-acco@bsides-sf-ctf-2019.iam.gserviceaccount.com");
            stringBuilder3.append("\",");
            a = stringBuilder3.toString();
            stringBuilder2 = new StringBuilder();
            stringBuilder2.append(a);
            stringBuilder2.append("\"client_id\": \"");
            stringBuilder2.append(BigInteger.valueOf(utils.gci()).multiply(BigInteger.valueOf(19)).multiply(BigInteger.valueOf(305363017965794407L)).toString());
            stringBuilder2.append("\",");
            a = stringBuilder2.toString();
            stringBuilder2 = new StringBuilder();
            stringBuilder2.append(a);
            stringBuilder2.append("\"auth_uri\": \"");
            stringBuilder2.append(new String(utils.ss("uggcf://nppbhagf.tbbtyr.pbz/b/bnhgu2/nhgu", 41)));
            stringBuilder2.append("\",");
            a = stringBuilder2.toString();
            stringBuilder2 = new StringBuilder();
            stringBuilder2.append(a);
            stringBuilder2.append("\"token_uri\": \"");
            stringBuilder2.append(new String(utils.ss("uggcf://bnhgu2.tbbtyrncvf.pbz/gbxra", 35)));
            stringBuilder2.append("\",");
            String stringBuilder4 = stringBuilder2.toString();
            StringBuilder stringBuilder5 = new StringBuilder();
            stringBuilder5.append(stringBuilder4);
            stringBuilder5.append("\"auth_provider_x509_cert_url\": \"");
            stringBuilder5.append(Utils.a("aHR0cHM6Ly93d3cuZ29vZ2xlYXBpcy5jb20vb2F1dGgyL3YxL2NlcnRz", 0));
            stringBuilder5.append("\",");
            stringBuilder4 = stringBuilder5.toString();
            stringBuilder5 = new StringBuilder();
            stringBuilder5.append(stringBuilder4);
            stringBuilder5.append("\"client_x509_cert_url\" : \"");
            stringBuilder5.append(Utils.a("aHR0cHM6Ly93d3cuZ29vZ2xlYXBpcy5jb20vcm9ib3QvdjEvbWV0YWRhdGEveDUwOS93ZWF0aGVyLWNvbXBhbmlvbi1zZXJ2aWNlLWFjY28lNDBic2lkZXMtc2YtY3RmLTIwMTkuaWFtLmdzZXJ2aWNlYWNjb3VudC5jb20=", 0));
            stringBuilder5.append("\"}");
            stringBuilder4 = stringBuilder5.toString();
            URL a3 = gVar.a(c.a(str2, str3).a(), TimeUnit.DAYS, com.b.c.c.g.a.a(i.a(new ByteArrayInputStream(stringBuilder4.getBytes()))));
            str4 = a3.toString();
            HttpURLConnection httpURLConnection = (HttpURLConnection) a3.openConnection();
            httpURLConnection.connect();
            httpURLConnection.getContentLength();
            BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(httpURLConnection.getInputStream()));
            StringBuffer stringBuffer = new StringBuffer();
            while (true) {
                str = bufferedReader.readLine();
                if (str == null) {
                    break;
                }
                StringBuilder stringBuilder6 = new StringBuilder();
                stringBuilder6.append(str);
                stringBuilder6.append("\n");
                stringBuffer.append(stringBuilder6.toString());
            }
            str = stringBuffer.toString();
        } catch (IOException e) {
            e.printStackTrace();
        }
        if (str4 != null) {
            Log.d("Background url", "Background URL in progress");
        }
https://binlog.usmannkhan.com/img/weathercompanion_file_tree.png    }

    /* access modifiers changed from: protected|final|synthetic */
    public final /* synthetic */ void onPostExecute(Object obj) {
        String str = (String) obj;
        Log.d("Complete reponse", str);
        try {
            JSONObject jSONObject = new JSONObject(str);
            str = ((JSONObject) jSONObject.get("display_location")).getString("city");
            jSONObject = (JSONObject) jSONObject.get("current_weather");
            String string = jSONObject.getString("temperature");
            String string2 = jSONObject.getString("precipitation");
            String string3 = jSONObject.getString("humidity");
            String string4 = jSONObject.getString("wind");
            ((TextView) this.a.findViewById(R.id.cityName)).setText(str);
            ((TextView) this.a.findViewById(R.id.precipitationValue)).setText(string2);
            ((TextView) this.a.findViewById(R.id.humidityValue)).setText(string3);
            ((TextView) this.a.findViewById(R.id.windValue)).setText(string4);
            ((TextView) this.a.findViewById(R.id.temperatureValue)).setText(string);
        } catch (JSONException e) {
            e.printStackTrace();
        }
    }
}
{% endhighlight %}

We can see that the bulk of the work here is assembling a string into a URL, making a request to it, and parsing the response into the weather data. The URL is assembled from a few parts: `dks()` and `ss()` both call out to C functions (that we have not decompiled or disassembled yet) via JNI, `String a` is a base64 encoded value that decodes to the first half of a private key, and `String a2` is another encoded value. After that there is an email address for a [GCP service account](https://cloud.google.com/iam/docs/service-accounts), and 2 ROT13 encoded URLs that decode to Google APIs authentication endpoints. It’s pretty clear that the bulk of this is service account credentials.

The two options at this point are:

1\. Reverse each of the functions that go into obfuscating the credentials.

2\. Log the assembled credentials after the `stringBuilder4 = stringBuilder5.toString();` line where it’s loaded into memory.

I went with the latter. I had a feeling that trying to recompile the obfuscated jadx output I had in front of me might be unnecessarily sadistic[^1] so I went up one level in the decompilation pipeline and used Apktool to only disassemble, not decompile, the apk. The big difference here is that after disassembly we don’t get java source code but smali, a human readable format of Dalvik bytecode.

The smali filetree is 1:1 with the java one we examined earlier with each java file having a smali equivalent. Unfortunately `a.smali` is about 6x the size of `a.java`. I searched the smali code to see what a call to the logging methods looks like and found this snippet:

```
const-string v1, "Background url"

const-string v2, "Background URL in progress"

invoke-static {v1, v2}, Landroid/util/Log;->d(Ljava/lang/String;Ljava/lang/String;)I
```

Focusing on the last line, a successful call to Log includes a reference to a String tag as the first argument and a log body as the second.

Next I looked for `"\"}"` which is the last thing appended to the credentials. I found the following smali code:

```
const-string v5, "\"}"

invoke-virtual {v6, v5}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;

invoke-virtual {v6}, Ljava/lang/StringBuilder;->toString()Ljava/lang/String;

move-result-object v5
```

and added the line `invoke-static {v2, v5}, Landroid/util/Log;->d(Ljava/lang/String;Ljava/lang/String;)I` beneath it. The last assignment to v2 was `"weather-companion"` so it should make a good tag.

{% highlight bash %}
$ apktool b weather-companion
I: Using Apktool 2.3.4
I: Checking whether sources has changed...
I: Smaling smali folder into classes.dex...
I: Checking whether resources has changed...
I: Building resources...
I: Copying libs... (/lib)
I: Building apk file...
I: Copying unknown files/dir...
I: Built apk...
$ keytool -genkey -v -keystore my-release-key.keystore -alias alias_name -keyalg RSA -keysize 2048 -validity 10000
Generating 2,048 bit RSA key pair and self-signed certificate (SHA256withRSA) with a validity of 10,000 days
	for: CN=Unknown, OU=Unknown, O=Unknown, L=Unknown, ST=Unknown, C=Unknown
[Storing my-release-key.keystore]
$ jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore my-release-key.keystore ./weather-companion/dist/weather-companion.apk alias_name
$ adb uninstall com.example.myapplication
Success
$ adb install ./weather-companion/dist/weather-companion.apk
{% endhighlight %}

You’ll notice that I uninstalled the app before installing the new version. If I hadn’t done that then Android would have noticed that the `com.example.myapplication` package on the device was signed by a different key than the new one and refuse to install it as a security measure. Alternatively, we could have changed the package name and have both installed at the same time.

{% highlight bash %}
$ adb logcat weather-companion:D \*:S
--------- beginning of system
--------- beginning of main
--------- beginning of crash
04-07 12:43:11.313 14365 14381 D weather-companion: {"type": "service_account","project_id": "bsides-sf-ctf-2019","private_key_id": "6dd7fc48a8b1d49edf7f03f74bc47713bec7d989","private_key": "-----BEGIN PRIVATE KEY-----\nMIIEvAIBADANBgkqhkiG9w0BAQEFAASCBKYwggSiAgEAAoIBAQCbNaJX7qZ2Sec4\n5W4ir+yYXJ3IwjZ8fwyw0PZSoiTb/iJqTcKk/ltjP61TrJHB5MqKm6Vz/WGw7GSm\nnd21xMFqclwuG8N7f+zhIK0XuvRBrS+cMEhw0RbHNUbcO3ZaghKffaLThzPY4x2r\nHh/N8lUYi4TB8WAGaAzCJH3pUi9rTG4+ucxO6pMNz3/ENzyOSmihR9Xb5OkMY/Aq\n7NJ7BAwH8oSWRlTllaC8Chl8wDdun6fKWgYYmmcBWXSjropu8f6MR7vMM3F0PEua\nYBz1ZpxVvYc1iiMS0WiNtAa1ZBuWhRenAiTfwOCt4rISfbgYSqCTSoK5tOWS52tM\ntuzt/OVjAgMBAAECggEAC9T230UuI25W1huHXdWTb7n/vUIw7SSyTvhfDsWVkb+5\n1+i9od5SESrVh79sDR/n4NEkt8blH5ulwJ3gPO8W34qARHORX2TNJgxbpad231rY\nekuj+hW2atFA6aEO0K+Bw+7L7twrs6j8pgLR4d1LZ2ebYz2HWHWuI06s2pCNVNyM\nSMn593YgfmzNotaoJ3dHAwKG8PuNRHh0qDqqLnu0574dXkTFkEePcedyzza007Iy\nCoNTUYJxDO7aJbTQPVyqm7mOewh1SYUdHSofcfALlC9eT2nn6+c9qRgGO3AwbC8V\ns9TmMR9+DWfHrUmPN8AKB4YKov/QGUv29svI0d8YwQKBgQDTobsDSI96xm2s+NnE\nPabZ+W76sg/1o1dPU4w4+/0u/RUT+vJoumsvwf5n7KUXVAP8npu6LYouNlHz9+zV\nThTINxyTrrA5VamFhoEpeY8OFboNVltxKj9nFvbS2jw2GLZQLapN0XIYE0axRNJs\nCSyc2LDYVVj0abjzx0CCFc0S+QKBgQC7v7kpSMJmIwl7FpJm/T9oakdvyZNzt3ZU\n+DTRmPXh/WM5c6j9vdzGKq3IlmHV/SSzVUfwQaxF8VzQlAi69fru/DStT8h7CpGd\nLEz8S0qq7ubbs7gODK/ZrwSQhv8doegZGu0ntURfaVL7AnyKGJVq2SLheVhMhJeZ\nm94mGME2OwKBgFXFSWcGRGhM/WxKGvAG0JWtGwZtnjw+rAcRZFZAApfFqIJFhXNe\ngkyDwhjadvpiaY87tP+ar1MVXteS1qCImbGfbGyKMw+5oQ/luHlXs9vQgGwhYMQX\njES6sOQ54IdIMrOCHnCVfzk0rsTvkJyKh1M2G05CIOBF7NiYG5PdRBT5AoGABEwT\nFNrReDz9DpApsanCNcWY9PoMIe3lC3TS4Kk7l3yRNNNs3sHlt7NqXtjyTE+K83/U\nMa+PHdq0YSHCQWU35RhorD7TO922D37gFDY080ychBLM96VasQTMefJdDHSUN17i\nZrJDaluixpP7/b0qTlPB9J8uYjH2tlFW+FBAu9kCgYBch8VvyBzlGay5r07joRju\ncdnVfrGAUdZRLxA5hrIaKvKFy28CaaKV/lZNI9NmdZntj6aIQ0rzUWidOQsmj2o4\n7VcArl5UNurhx4uhg2GClcUIHY5YdExHfEujBu6uMQdZjBpX87uoaggC7jwJanQC\nlApTPc3478g/mho9S5eDMw==\n-----END PRIVATE KEY-----\n","client_email": "weather-companion-service-acco@bsides-sf-ctf-2019.iam.gserviceaccount.com","client_id": "116037946827001874660","auth_uri": "https://accounts.google.com/o/oauth2/auth","token_uri": "https://oauth2.googleapis.com/token","auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs","client_x509_cert_url" : "https://www.googleapis.com/robot/v1/metadata/x509/weather-companion-service-acco%40bsides-sf-ctf-2019.iam.gserviceaccount.com"}
{% endhighlight %}

Loading the app and filtering the logs to the `weather-companion` tag spits the key right out! Using a similar smali trick to log the generated URL we can see that it’s a signed request to `https://storage.googleapis.com/weather-companion/weather.json`, a GCS bucket named `weather-companion`.

I authenticated the [`gcloud`](https://cloud.google.com/sdk/gcloud/) tool using my new service account credentials and started to poke around GCS. After a quick detour where we and the organizers realized that the flag wasn’t searchable 😅 all it took was a simple `ls` on the bucket!

{% highlight bash %}
$ gcloud auth activate-service-account weather-companion-service-acco@bsides-sf-ctf-2019.iam.gserviceaccount.com --key-file=weather_priv_key.json
$ gsutil ls gs://weather-companion
gs://weather-companion/flag.txt
gs://weather-companion/weather.json
{% endhighlight %}

🎉

[^1]: While writing this post I decided to give this method a shot. I gave up quickly.