---
layout: single
title:  "SecureishShell – 0xL4ugh CTF"
excerpt: "SecureishShell is a bit different to write about, since I built it. My goal is to introduce something that i rarely see in challenges, which is Keymap walking passwords..."
classes: wide
gvt: of0LcSklyF8S5vFI5P_r8_OkmpfZ6SX5QuEQbLRDS7g
---
![1](/assets/images/SecureishShell/1.jpg)

SecureishShell is a bit different to write about, since I built it. My goal is to introduce something that i rarely see in challenges, which is Keymap walking passwords


## Machine Overview

The box starts by finding a file on port 80 talking about a used `(Phoenix)`{:style="color:orange"} who thinks that Keymap walking passwords are safe. also you can find phoenix's password hash on memcached, from there you'll have to use kwprocessor to generate a wordlist that can crack that hash. After cracking the hash you get a SSH shell on the machine, and find that phoenix can execute `tee`{:style="color:orange"} as another uesr, but there is a trick to it. after you successfully use you private key to login as Jett you see that she has the ability to install NodeJs packages, you use that to install a fake Node App that gets you a shell as Brimstone 

## Recon

`nmap` shows a webserver, memcached and ssh open on the target
```py
   root@kali:~# nmap -p- -sT --min-rate 10000 52.188.23.13
   Starting Nmap 7.80 ( https://nmap.org ) at 2021-01-16 03:52 EST
   Nmap scan report for 52.188.23.13
   Host is up (0.058s latency).

   PORT      STATE SERVICE
   22/tcp    open  ssh
   80/tcp    open  http
   11211/tcp open  memcache
```

## TCP 80 - HTTP

The site seems to have the default apache page but running `fuff`{:style="color:orange"} will reveal a `/robots.txt`{:style="color:orange"}

![2](/assets/images/SecureishShell/2.png)

## Keymap Walking Passwords

the image from the webserver contain a text file embedded into it that can be extracted with `steghide`{:style="color:orange"} but that was hidden there as a troll so i'll disregard it for now. On the other hand, `note.txt`{:style="color:orange"} mentions `Keymap Walking Passwords`{:style="color:orange"}.

![3](/assets/images/SecureishShell/3.png)

Keyborad walk is when you start with a specific key on the keyboard and then pick a direction (or multiple directions) and start hitting keys as you `walk`{:style="color:orange"}.

`kwprocessor`{:style="color:orange"} creators say, and i quote, 
<blockquote>
 Sometimes people come to the conclusion that a keyboard-walk is such a "Medium" security solution. A password like "q2w3e4r" comes to the rescue, it looks random for those who do not crack passwords and random is good
</blockquote>

![en_sawtooth](/assets/images/SecureishShell/en_sawtooth.png)


What makes keymap walking so successful (until now) is that an attacker would need to know the starting key, direction, direction changes, if any special key is used and when, and of course the ending key.
## TCP 11211 - Memcached

Well... [KwProcessor](https://github.com/hashcat/kwprocessor){:target="_blank"} is a tool that makes creating keymap walking wordlists very easy to do, but still we need to find a hash to crack and here is when memcached comes in.

If you are not familiar with memcached, [here is a good atricle](https://www.hackingarticles.in/penetration-testing-on-memcached-server/){:target="_blank"} from Hacking Articles that explains how to pentest it.

Basically, `Memcached`{:style="color:orange"} is a distributed memory object caching system. It is an in-memory key-value store, and by in-memory i mean that if you restart the service then all the data on it will be lost. It caches data and makes it quickly available so that an application doesn’t have to re-query a database over and over again.

`stats slabs`{:style="color:orange"} gives information about the various slabs. In this case, there’s only one in use (4)
Then I can see what’s in the cache with `stats cachedump x y`{:style="color:orange"}, where `x`{:style="color:orange"} is the slab number I want and `y`{:style="color:orange"} is the number of keys I want to dump (0 = all).

![4](/assets/images/SecureishShell/4.png)

## Cracking the hash / Getting Shell as Phoenix

Now that i have a hash i can start by generating the wordlist. since my goal here was not to tourture you with cracking i left clues in the note like `Keymap walking password`{:style="color:orange"}. Googling this keyword should land you on [this blog](https://cyberarms.wordpress.com/tag/password-cracking/){:target="_blank"} that discusses hot to create Hashcat Keymap Walking Password Wordlists. I strongly recommend reading this article to get a better explaination on how to create a wordlist

I used the simplest KWP wordlist to choose a password from. to do so you'll have to install `KwProcessor`{:style="color:orange"} then run:

```bash 
./kwp basechars/full.base keymaps/en-us.keymap routes/2-to-10-max-3-direction-changes.route > kwp.txt
```		
I then saved the hash and fed it `john`{:style="color:orange"} and the hash got cracked in under 15 seconds

![5](/assets/images/SecureishShell/5.png)

now i can simply login via SSH and get shell
## Escalating to Jett

logically, one of the first thing you check when you get a new user shell is running `sudo -l`{:style="color:orange"}, by doing so you'll notice that Phoenix has the ability to run `tee`{:style="color:orange"} as Jett.<br />
`tee`{:style="color:orange"} reads the standard input and writes it to the standard output and one or more files. It basically takes the output of a command and do both, have it displayed and saved in a file.

![6](/assets/images/SecureishShell/6.png)

**PS:** That asterisk symbol (*) at the end is just a wildcard

Instinctly, one should think that he can use that to write their public key in jett's authorized_keys using something like:<br />
`echo 'PublicKeyHere' | sudo -u jett /usr/bin/tee -a /home/jett/.ssh/authorized_keys`{:style="color:orange"}, but trying that will not work.

## AuthorizedKeysFile
The main goal while designing this challenge is to illustrate that `Obscurity is not Security`{:style="color:orange"}. So what i did was manipulating SSH configurations. i manipulated a parameter called `AuthorizedKeysFile`{:style="color:orange"} and changed it's value to `AuthorizedKeysFile UsePAM`{:style="color:orange"}. I purposely did name it this way so that it can blend in with the rest of the configs.

![7](/assets/images/SecureishShell/7.png)

`AuthorizedKeysFile UsePAM`{:style="color:orange"} is actually defining file that can hold public keys. by default it looks for that file in user's home directory. The default value for this parameter is set to: 

![authkey](/assets/images/SecureishShell/authkey.png)

That is why when we add our public keys we add it in a file named `authorized_keys`{:style="color:orange"} inside a dir named `.ssh`{:style="color:orange"}

The concept is still the same, it is just the file name that is changed. so i can modified my previous command and got a shell as Jett:
```bash
echo 'PublicKeyHere' | sudo -u jett /usr/bin/tee -a /home/jett/UsePAM
```

![8](/assets/images/SecureishShell/8.png)

## Installing a Malicious NodeJs App / Getting Shell as Brimstone

The next step also stars by running `sudo -l`{:style="color:orange"} to find out that you can run `/usr/bin/npm i`{:style="color:orange"} as brimstone. where `i`{:style="color:orange"} is an aliase for `install`{:style="color:orange"}.

![9](/assets/images/SecureishShell/9.png)

The idea of this challenge is inspired by [this blogpost](https://github.com/joaojeronimo/rimrafall){:target="_blank"}. I recommend reading it to understand the issue.<br />
The goal of this challenge is to show you how npm can be dangerous.

The idea is that a NodeJS package is defined in a file named `package.json`{:style="color:orange"}. Inside that file there’s an item called `scripts`{:style="color:orange"} with a child called `preinstall`{:style="color:orange"}, `preinstall`{:style="color:orange"} is a command that will get executed before the package is installed.

I'll create my `package.json`{:style="color:orange"}, `npm`{:style="color:orange"} requires that a package must have a name and a version so i'll specify that and the rest of the items from the repo can be neglected

![10](/assets/images/SecureishShell/10.png)

The only thing left is running it and i get a shell and can grab the flag

![11](/assets/images/SecureishShell/11.png)

<img src="http://www.hackthebox.eu/badge/image/206208" alt="Hack The Box">