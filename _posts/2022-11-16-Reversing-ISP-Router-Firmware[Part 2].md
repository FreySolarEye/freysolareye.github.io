---
title: "Reversing ISP Router Firmware [Part 2]"
date: "2022-11-16"
header:
  teaser: /assets/images/RE/IoT_Base.png
  teaser_home_page: true
tags: reverse,firmware
categories: RE
toc: true
toc_sticky: false
toc_label: "Table of Contents"
---

It's been a year since i have posted the <a href="https://freysolareye.github.io/re/Reversing-ISP-Router-Firmware-Part-1/" > **Part[ 1 ]**</a> of the Reverse Engineering series mainly because iam in the military; oh well anyway, the  <a href="https://freysolareye.github.io/re/Reversing-ISP-Router-Firmware-Part-2/" > **Part[ 2 ]**</a>  will be focusing on emulating the firmware. We will be using some out of the box tools to make our lives and debugging easier. Note to the viewers  <a href="https://freysolareye.github.io/re/Reversing-ISP-Router-Firmware-Part-2/" > **Part[ 2 ]**</a> highlights the emulation therefore it's assumed that every tool and firmware that it's being shown here is downloaded & installed.

# Disclaimer 
{:.no_toc}

This is strictly for educational purposes <b>ONLY</b> and not to be used for conducting any illegal activities. I hold no responsibility for misuse of this information. i was questioned by my  colleagues if i got permission from my ISP to Reverse Engineer their firmware well no duh; There's no reason to ask for permission for a firmware version which is more likely  outdated  and deprecated since 2013 or so. 

# Requirements
{:.no_toc}

  * Router Firmware [ ZTE H108NS ]
  * A Linux/Unix Computer or Virtual Machine
  * An installed copy of ``FirmAE``

# What is FirmAE 
{:.no_toc}

FirmAE is a fully-automated framework that performs emulation and vulnerability analysis. FirmAE significantly increases the emulation success rate (From Firmadyne's 16.28% to 79.36%) with five arbitration techniques.

Download Link : <a href="https://github.com/pr0v3rbs/FirmAE" > FirmAE</a>

# Using FirmAE to perform emulation 
{:.no_toc}

What we need to do first is use FirmAE to check if we can emulate the firmware, to do that : 

    1. cd /opt/FirmAE
    2. sudo ./run.sh -r ZTE tclinux.bin

<img src="{{ site.baseurl }}/assets/images/RE/2022-11-16_00-36.png">


As we can see the emulation works perfectly and we're granted with a Network Interface  which leads to a web page asking as for Basic Auth credentials. Therefore we're one step closer to view the router web panel. Before we can do that we need the credentials but we don't have them so we need to dig a little deeper into the firmware from the inside in order to uncover or better; Manipulate some things to get past the Basic Auth. 


# Debugging the Firmware
{:.no_toc}

We have emulated the firmware but now we need to debug it, how are we gonna do that; Well FirmAE has another tool which can help us to directly connect to the current qemu session. What we need to do is list our qemu session, by default every qemu session is under **/tmp/** directory so it won't be hard to locate the number of the session. 

Let's execute the needed commands : 

    1. cd /opt/FirmAE/
    2. ls -la /tmp/ |grep qemu 
    3. sudo ./debugger.py 1

In my case the qemu session is on number 1, after executing it by default the script will try to  connect to the session with Netcat, but it will fail. In order to get a solid shell inside we need to choose the option **Socat** so input the number **1** 

And finally we have a **/bin/sh** shell inside 

<img src="{{ site.baseurl }}/assets/images/RE/2022-11-16_01-13.png">


# Debugging Romfile.cfg
{:.no_toc}

Romfile.cfg is a piece of file that holds the basic configuration of the Router, latest firmwares encrypt this file on the runtime so third party operatos  would never be able to achieve reading the none encrypted version of it without having the  original extracted firmware or a flash dump directly from the Router ROM. From our point of view the **romfile.cfg** is unencrypted and we can view the contents of it directly.

  <img src="{{ site.baseurl }}/assets/images/RE/2022-11-16_01-56.png">

Theoritical point of view; Under the path of **/userfs/romfile.cfg** there's our file, grepping the file and viewing it's contents we can assume that the credentials embedded there are the web panels basic auth credentials.


Command : 
        
        cat romfile.cfg |grep username

        <Entry0 username="admin" web_passwd="poqv7069"

But testing them on the Basic Auth web panel of the router shows that they're incorrect which is kinda odd; Because clearly it states that the credentials are for **web_passwd** entry. Never the less after some time trying to make sense of it i tried to approach it with a different methodology. Instead  we can just tweak some mechanisms to work for our advantage in order to access the router web portal without breaking the Basic Authentication module.


# What is Βoa
{:.no_toc}

A little bit information about boa, Boa is a discontinued since 2005 open-source small-footprint web server that is suitable for embedded applications. To simplify  things up it's just a web based client which serves basic content with minimal security policies.

# Debugging Boa Configuration
{:.no_toc}

Now that we're inside the Router environment we can take a look how Boa is being executed and what configuration does it parses, to do that we're gonna list our current processes and grep through our way to locate Boa process.

On the router shell environment let's execute the below command : 

    1. ps auxf |grep boa
 

  <img src="{{ site.baseurl }}/assets/images/RE/2022-11-16_15-53.png">

  As it suggests Boa seems to use the config under the /boaroot and is being executed with the flag of **-d** which suggests that it instruct Boa not to fork itself (non-daemonize). Taking a look at the boa configuration we can see that  the basic Authorization is being parsed via **/etc/passwd**

    # Auth: HTTP Basic authorization. Format is "Auth <Directory> <PasswdFile>".
    # Password file should be readable _ONLY_ by root or trusted user(s). This file
    # is opened before boa gives out privs.
    # Example: Auth /secret /var/www/secret.passwd

    #Auth /  /etc/passwd

# Manipulating Basic Authentication Mechanism
{:.no_toc}

  Viewing the contents of **/etc/passwd**  clearly states that there's a user called **admin** which has a valid **/bin/sh** shell and permissions as **UID 0** which leads to root permission access. Here's the output of the **/etc/passwd**

      admin:$1$$UwJbejuDC6tr4MjtAZGfV0:0:0:root:/:/bin/sh                                                                           

  After hours of bashing my head around trying to crack the salted MD5 password, i thought why would i try to crack it when i can instead re-generate a new password for the user **admin** without knowing his old password; Remember i have root permissions so changing it is easy.

  Let's change the password by executing the below commands : 

    1. passwd admin
    2. Input a new password (used admin as password from my side)
    3. Re-enter the password for validation

  <img src="{{ site.baseurl}}/assets/images/RE/2022-11-16_16-18.png">

  Now recaping back, we know Boa http server parses the basic authentication based on the **/etc/passwd**. Previously we didn't know the password, but  now we do let's see if Boa re-checked the **/etc/passwd** and refreshed his configuration with our newly created password, so let's go back to the router Web Panel and input the **username** and **password**

  <img src="{{ site.baseurl }}/assets/images/RE/2022-11-16_16-28.png">

  And just like that we're inside the router web panel. Neat stuff hu?

<p align="center">
<img src="{{ site.baseurl }}/assets/images/RE/magic-henning.gif">
</p>

## Keynotes
{:.no_toc}

1. Always perform Dynamic / Static analysis on the firmware beforehand
2. Taking a look around  and list current proccesses may give you a good insight on how the underlying system works
3. Thinking outside of the box and tinkering with software mechanisms when it's necessary

# Summary
{:.no_toc}

We've successfully emulated our router firmware using out of the box tools, perfomred dynamic analysis inside the emulated running state of the firmware. Thinked out of the box techniques and applied them to tinker with mechanisms and finally getting access inside the Router Web Portal. On <a href="Part[3]" > Part[ 3 ]</a> we will be talking on how we can find and analyse vulnerabilities or who knows fuzzing for some Bofs too;