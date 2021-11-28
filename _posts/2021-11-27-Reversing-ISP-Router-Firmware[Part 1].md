---
title: "Reversing ISP Router Firmware [Part 1]"
date: "2021-11-27"
header:
  teaser: /assets/images/RE/IoT_Base.png
  teaser_home_page: true
tags: reverse,firmware
categories: RE
---

A few days ago i decided that i want to reverse engineer one of my ISP's routers, specifically the ZTE Router Models. Why would i do that? Well because it's fun messing around with custom firmwares. 

**Disclaimer**

This is strictly for educational purposes <b>ONLY</b> and not be used for conducting any illegal activities. I hold no responsibility for misuse of this information.


**Requirments**

  * Router Firmware [ ZTE H108NS ]
  * A Linux/Unix Computer or Virtual Machine
  * An installed copy of ``binwalk``
  * Optional: John the Reaper, Hashcat

**Download the Firmware**

First, we need to download the firmware to begin reverse engineering it.

Download Link : <a href="https://help.cosmote.gr/system/templates/selfservice/gnosisgr/Files2/ZTE_H108NS_CONNX_PSTN_V107u_ZRD_GR2_A68_20150720_tclinux.rar" > ZTE-H108NS</a>

**Digging into the Firmware** 

Now that we have downloaded it let's make a workplace for it : 

    1. mkdir -p /opt/firmware/
    2. cp /home/<user>/Downloads/ZTE_H108NS_CONNX_PSTN_V107u_ZRD_GR2_A68_20150720_tclinux.rar .
    3. unrar x ZTE_H108NS_CONNX_PSTN_V107u_ZRD_GR2_A68_20150720_tclinux.rar
    4. The unzipped files should be : tclinux.bin


<img src="{{ site.baseurl }}/assets/images/RE/2021-11-27_23-43_ls.png">




First things first we need to run ``binwalk`` on our .bin fimrware image: 

<img src="{{ site.baseurl }}/assets/images/RE/2021-11-27_23-51_binwalk.png">

As you can see, this image consists of two parts: An LZMA compressed file (compare to a .zip or .rar file) and a SquashFS Filesystem, common for compressing a Linux or Unix OS into small space. It's important to make a note that the LZMA file starts at byte 256 (or 0x100 in hexadecimal). While the SquashFS filesystem is located at byte 955740 (or 0xE955C in hexadecimal).

**Getting that SquashFS**

Before we get the SquasFS we need to install sasquatch, so we can automate the procedure by combining it with ``binwalk`` :

 {% highlight bash %}
   1. git clone https://github.com/devttys0/sasquatch.git
   2. sudo apt-get install build-essential liblzma-dev liblzo2-dev zlib1g-dev
   3. sudo ./build.sh

{% endhighlight %}

 After everything is ready, we can finally execute binwalk with the help of sasquatch.
 
Executing the command : ``binwalk --extract tclinux.bin -1``

 <img src="{{ site.baseurl }}/assets/images/RE/2021-11-28_00-36_binwalk_preserve.png">

  And finally we can see that we have successfully extracted the squashFS system


<img src="{{ site.baseurl }}/assets/images/RE/2021-11-28_00-38_squash_ex.png">

  A little bit explanation on what does the command argument ``-1`` does on binwalk, the specific argument allows us to extract every symlink that it's supposed to be inside of the  squashFS system. By default it does not extract the symlinked files for obvious reasons.


  **And now what? Is there something else there?**

  Now that you have pretty much full access to the whole squasFS system you can tinker it as you please, what i would do in my case is to look for default credentials on services, or files and then i would setup a testing environment with  <b>firmadyne</b> and emulate the Router's firmware in order to find vulnerabilities, or bypass restrictions. Υou can also use the <b>mod-firmware-kit</b> to re-pack the whole image file with a backdoor or tweak it as you like.

  Resources for both of the tools : 

  Firmadyne : <a href="https://github.com/firmadyne/firmadyne" target="_blank" > Firmadyne</a><br>
  Firmware-Mod-Kit : <a href="https://www.kali.org/tools/firmware-mod-kit/" target="_blank" > Firmware-Mod-Kit</a> 


  **Summary**

  It was fun reversing this one, as it was easy and straightforward, on  <b>Part[2]</b> we will focus on how we can setup a testing environment with firmadyne and emulate our extracted firmware.