---
layout: single
title:  "CyCTF 2024 Finals - Android Challenges"
classes: wide
excerpt: CyCTF is maintained by the CyShield and the 2024 edition happened on the 20/11/2024....
categories: CTF
toc: true
---
## Introduction
CyCTF is maintained by the [CyShield](https://cyshield.com/) and the 2024 edition happened on 20/11/2024.

As usual, this was a great CTF with some really nice Android challenges. I've managed to solve 2 Android challenges `Armor`{:style="color:orange"} and `Andoroido`{:style="color:orange"}, and I thought since they were such fun challenges they deserved a writeup.

## Armor
Starting by taking a quick look at the app on my emulator, It asks for a `Pin Code`{:style="color:orange"}, and If provided incorrectly, the app will crash. 

![0](/assets/images/CyCTF2024/0.png){: width="500" height="600"}

The next step is to open the APK in a decompiler and start to analyze how the application works. It seems that the `Pin Code` is being passed to the function named `CP`{:style="color:orange"} on `line 27`, this function is defined inside the native library, and despite the name of the library is encrypted you can notice that there is only one native library shipped with the APK (`libarmor.so`{:style="color:orange"}) so no need to guess which one is it here. 

![1](/assets/images/CyCTF2024/1.png){: width="800" height="700"}

To analyze the code of the `CP`{:style="color:orange"} function, I dropped the library in IDA and after looking at the code, it appears to be pretty straightforward. The code copies the provided `Pin Code` to the local variable `v4`{:style="color:orange"} on line 9, it then copies it to `v7`{:style="color:orange"} on line 12, and finally it is passed to `sub_6A7EC`{:style="color:orange"} on line 13 where it basically compares it with the string `534925`{:style="color:orange"}.

![2](/assets/images/CyCTF2024/2.png){: width="800" height="700"}

![3](/assets/images/CyCTF2024/3.png){: width="800" height="700"}

Finally, trying this `Pin Code` in the application will reveal a Toast message which eventually turns out to be the encrypted flag in base64 encoded format. 

![4](/assets/images/CyCTF2024/4.png){: width="800" height="700"}

After getting the flag, I started digging deeper into the code and noticed that inside the `Prompt`{:style="color:orange"} class lies the code for the toast message that printed the encrypted flag. After that, the application uses the [DexClassLoader](https://developer.android.com/reference/dalvik/system/DexClassLoader) class to load a certain class and invoke a function from within that class. You can also notice on `line 64` that the location from which it'll load the code is inside the app's cache directory. Furthermore, it prints something in the logs so it is worth checking that out as well. 

![5](/assets/images/CyCTF2024/5.png)

The logs of the application point to a dex file inside the cache directory which is consistent with what is on `line 64`.

![6](/assets/images/CyCTF2024/6.png)

I grabbed a copy of that dex file and noticed that it is a class named Utils which is used for encryption and decryption as well. You can notice that it uses AES with CBC mode. It uses a IV shown on `line 55` and the key is defined with a `null`{:style="color:orange"} value at the very beginning of the class. Finally, notice how all the class code is readable except for the code of the constructor. 

![7](/assets/images/CyCTF2024/7.png)

To understand why jadx fails to decompile the constructor you'll have to look at the small code. The code starts by checking the length of the parameter sent to the constructor and makes sure it is 32 bytes in length. The issue starts from `line 52` where you can see an unconditional jump to label `goto_e3`{:style="color:orange"}. 

![8](/assets/images/CyCTF2024/8.png)

If you scroll down to `goto_e3`{:style="color:orange"}, you can see the pattern from line 313 to 317 is basically an infinite loop that hindered jadx from decompiling the constructor properly. 

![9](/assets/images/CyCTF2024/9.png)

I removed the jump on `line 52` and removed the infinite loop pattern as well and rebuilt the dex file again and this time you can see the code and the bytes of the encryption key.

![10](/assets/images/CyCTF2024/10.png)

Finally, after compiling all the parts of the key, you can decrypt the flag with a simple CyberChef formula. 

![11](/assets/images/CyCTF2024/11.png)

---

## Andoroido

Andoroido is my personal favorite in all of the challenges since it is about Android Internals which is a topic I've been very interested in lately. 

After inspecting the application in jadx, it is very clear that the main activity (which is exported by default) takes an extra called `url`{:style="color:orange"} and if the url variable is set, the app calls a function named `downloadDex`{:style="color:orange"}.

![12](/assets/images/CyCTF2024/12.png)

The `downloadDex`{:style="color:orange"} function simply downloads whatever the url is pointing to and saves it in the applications `Files` directory under the `name payload.dex`{:style="color:orange"}.

![13](/assets/images/CyCTF2024/13.png)

Next, `downloadDex` calls a new function named `loadDex`{:style="color:orange"} which again uses the [DexClassLoader](https://developer.android.com/reference/dalvik/system/DexClassLoader) class to invoke a function named `execute`{:style="color:orange"} inside a class named `com.clone.payload.Main`{:style="color:orange"}. Finally, `loadDex`{:style="color:orange"} checks the package name and if it is equal to `l337_P0wer`{:style="color:orange"} It'll print the flag.

![14](/assets/images/CyCTF2024/14.png)

This last part is basically the entire challenge because the `getPackageName`{:style="color:orange"} function will always return `com.cyctf.andoroido`{:style="color:orange"}. 

To move the easier stuff out of the way, below is the code needed to communicate with `Andoroido` and sends it the url which points to the malicious dex file. Just a pretty straightforward Intent with an Extra, nothing fancy.

![15](/assets/images/CyCTF2024/15.png)

The final part is about modifying the package name and for that, let me quickly summarize a few key pieces of information needed to perform this trick.

1. `ActivityThread`{:style="color:orange"}: It is a critical Android internal class that manages the main thread of an application.
2. `mBoundApplication`{:style="color:orange"}: It is a field that contains an `AppBindData`{:style="color:orange"} instance, which holds information about the current app.

    ![16](/assets/images/CyCTF2024/16.png)

3. `LoadedApk`{:style="color:orange"}: This class contains the `mPackageName`{:style="color:orange"} field. Modifying this field will change the package name and ensure changes will propagate to runtime checks that reference it. for example checks like the `getPackageName()`{:style="color:orange"} function used in the challenge.


![17](/assets/images/CyCTF2024/17.png)

So, to put it all together, I simply need to modify the value of `mBoundApplication.info.mPackageName`{:style="color:orange"}. Since this code will be loaded from a `Dex`{:style="color:orange"} file using the keyword `this`{:style="color:orange"} turned out to be risky. `this`{:style="color:orange"} in this context refers to the class in which `execute`{:style="color:orange"} is defined, not the actual Application instance or its related class loader. To avoid this, you can see on lines 8 and 9 I'm using The `ActivityThread`{:style="color:orange"} singleton to retrieve the `currentActivityThread()`{:style="color:orange"} method. This is essential for modifying the internal app's data.

![18](/assets/images/CyCTF2024/18.png)

Before compiling this application, one final modification needs to be made. I needed to set the `multiDexEnabled`{:style="color:orange"} field to `false`{:style="color:orange"} in the gradle configurations to avoid dividing the code into multiple dex files. 

![19](/assets/images/CyCTF2024/19.png)

Finally, compile the code, extract the Dex file, host it on your server, and open the exploit application from earlier and you'll be treated with the flag. 

![20](/assets/images/CyCTF2024/20.png)

