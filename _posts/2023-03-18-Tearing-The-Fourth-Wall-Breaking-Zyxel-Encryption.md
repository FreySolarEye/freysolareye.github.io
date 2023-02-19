---
title: "Tearing the Fourth Wall : Breaking Zyxel Encryption"
date: "2023-02-18"
header:
  teaser: /assets/images/RE/IoT_Base.png
  teaser_home_page: true
tags: reverse,firmware
categories: RE
toc: true
toc_sticky: false
toc_label: "Table of Contents"
---

During a web application penetration test assessment for a client, I came across a Zyxel ZyWall 310 web portal. At the time, I couldn’t exploit it, nor could I find any weaknesses in it, but for some reason, it stuck in my head. I thought that I wanted to poke around and dissect the firmware. This blog describes my train of thought, how I approached it, and what techniques I applied to bypass the encryption mechanism of the stock firmware to get the uncompressed version of it. As far as I know, this topic has been analyzed in plenty of research papers already. Shout out to **@jaylagorio** and **@Dr Amir Mehmood** for the awesome research they’ve done so far.

# Disclaimer 
{:.no_toc}

This is strictly for educational purposes <b>ONLY</b> and not to be used for conducting any illegal activities. I hold no responsibility for misuse of this information. 

# Requirements
{:.no_toc}

  * Zyxel 310 Firmware [ 4.73(AAAB.0 ]
  * A Linux/Unix Computer or Virtual Machine
  * A Brain 

# Downloading the Firmware
{:.no_toc}

In order to download the firmware we need to make an account at : <a href='https://portal.myzyxel.com'>https://portal.myzyxel.com</a>  

Download Link : <a href='https://portal.myzyxel.com/my/firmwares?fw_version=4.73(AAAB.0)C0&model=ZyWALL%20310'> Zyxel_310_Firmware_4.73(AAAB.0)</a>

<img src="/assets/images/RE/Zyxel/ZY_DOWNLOAD.png">


# Firmware Contents 
{:.no_toc}

If you downloaded the abovementioned version you'll find yourself with a file named **firmware.zip** unzipping it  unveils the below files :

<img src="/assets/images/RE/Zyxel/ZY_FILES.png">


# Signature Checks & Validation

To begin with we need to understand that we're dealing with the  encrypted binary **473AAAB0C0.bin**  To be able to tell what's the encryption and file type we need to examine carefully the starting Hex Bytes, and validate them with the file type signature list. Using hexeditor on the **473AAAB0C0.bin** file we have the below starting Hex Bytes : **50 4B 03 04** looking them on the signature list here : <a href='https://en.wikipedia.org/wiki/List_of_file_signatures' target="_blank" > Signature<a/> which concludes that the file type is PK which is 7z compressed file type.

<img src="/assets/images/RE/Zyxel/ZY_Signature.png">

Now if someone tries to actually unzip the **473AAAB0C0.bin** a pop up will occur asking the users to input the password.


# Static Analysis 

In the PDF documents distributed with the firmware, a recovery method is described in the event that the upgrade procedure is unsuccessful. Taking a look here <a href='https://support.zyxel.eu/hc/en-us/articles/360014273779-Firmware-Recovery-Process-on-USG-ATP-VPN-Firewalls' target="_blank"> Recovery Manual<a/> we can assume that the Zyxel is actually using some short of tool to achieve the recovery of the firmware.

Based on the information in the low level recovery guide,  we can assume that the **"473AAAB0C0.ri"** file is directly executed by the system to install the fundamental components in order to then be able to flash the firmware from the **".bin"** file.

I have extracted the contents of **"473AAAB0C0.ri"** below : 

```zsh
┌──(frey㉿frey)-[~/Desktop/Projects/Zyxel_310]
└─$ binwalk -e 473AAAB0C0.ri -1

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
512           0x200           uImage header, header size: 64 bytes, header CRC: 0xE1545E1D, created: 2022-11-17 17:07:29, image size: 4016640 bytes, Data Address: 0x5000000, Entry Point: 0x80CC9870, data CRC: 0x68BE715C, OS: Linux, CPU: MIPS, image type: OS Kernel Image, compression type: lzma, image name: "Linux Kernel Image"
576           0x240           LZMA compressed data, properties: 0x5D, dictionary size: 8388608 bytes, uncompressed size: 11195488 bytes
```

And the files inside of the 240 binary : 

```zsh
┌──(frey㉿frey)-[~/Desktop/Projects/Zyxel_310/_473AAAB0C0.ri.extracted]
└─$ binwalk -e 240 -1          

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             ELF, 64-bit MSB MIPS32 rel2 executable, MIPS, version 1 (SYSV)
5107864       0x4DF098        Linux kernel version 3.10.8
5179656       0x4F0908        gzip compressed data, maximum compression, from Unix, last modified: 1970-01-01 00:00:00 (null date)
5196240       0x4F49D0        Unix path: /home/sdd1/buildbot/473-share1100-k3-slave/473-share1100-k3/build/share1100-r106233-k3/share1100/src/kernel
5284016       0x50A0B0        DES SP2, big endian
5284528       0x50A2B0        DES SP1, big endian
5824472       0x58DFD8        Unix path: /usr/bin/magic-seed
5853384       0x5950C8        Unix path: /home/sdd1/buildbot/473-share1100-k3-slave/473-share1100-k3/build/share1100-r106233-k3/share1100/src/kernel/arch/mips/include/as
5939112       0x5A9FA8        xz compressed data
6040223       0x5C2A9F        Copyright string: "Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>"
6092400       0x5CF670        Neighborly text, "NeighborSolicits"
6092424       0x5CF688        Neighborly text, "NeighborAdvertisementsorts"
6097330       0x5D09B2        Neighborly text, "neighbor %.2x%.2x.%pM lost rename link %s to %s"
6463872       0x62A180        CRC32 polynomial table, little endian
6643248       0x655E30        Unix path: /usr/local/zld_udev/sbin/uevent_helper.sh
7060288       0x6BBB40        Flattened device tree, size: 7847 bytes, version: 17
7068160       0x6BDA00        Flattened device tree, size: 12032 bytes, version: 17
7085520       0x6C1DD0        ASCII cpio archive (SVR4 with no CRC), file name: ".", file name length: "0x00000002", file size: "0x00000000"
7085632       0x6C1E40        ASCII cpio archive (SVR4 with no CRC), file name: "init", file name length: "0x00000005", file size: "0x0000000D"
7085764       0x6C1EC4        ASCII cpio archive (SVR4 with no CRC), file name: "zyinit", file name length: "0x00000007", file size: "0x00000000"
7085884       0x6C1F3C        ASCII cpio archive (SVR4 with no CRC), file name: "zyinit/zld_fsextract", file name length: "0x00000015", file size: "0x00017098"
```

What interests us the most are the files of :

```zsh
7085764       0x6C1EC4        ASCII cpio archive (SVR4 with no CRC), file name: "zyinit", file name length: "0x00000007", file size: "0x00000000"
7085884       0x6C1F3C        ASCII cpio archive (SVR4 with no CRC), file name: "zyinit/zld_fsextract", file name length: "0x00000015", file size: "0x00017098"
```

Looking at the extracted files, it is possible to identify the **"zyinit"** binary, which contains some firmware related strings.

By analyzing it, it is possible to see that it launches other external commands, in particular the **"zld_fsextract"** command:


I've loaded the binary into Ghidra just to analyze the routine as it states below : 

<img src="/assets/images/RE/Zyxel/ZY_INSTRUCT.png">

Taking as a note the procedure starts on the instruction code **10006ffc** and finishes at **10007010** via a mnemonic bal (ARM - Branch and Link Instruction to call a function and moves the link register to the PC (MOV PC, LR) to return from a function in our case the function is **FUN_10025568** which is the exit loop of the sequencial order of the IF  statement)

<img src="/assets/images/RE/Zyxel/ZY_BAL.png">


Searching online for these executable binaries, only an interesting URL was identified <a href='https://www.dslreports.com/forum/remark,26961186' target="_blank"> https://www.dslreports.com/forum/remark,26961186 <a/>  which gave me some information about the ZIP password:

<img src="/assets/images/RE/Zyxel/ZY_PASSWORD.png">

Searching in the **"zld_fsextract"** binary with strings for the password, it is possible to identify the actual usage of the file : 

```zsh

┌──(frey㉿frey)-[~/…/Projects/Zyxel_310/_473AAAB0C0.ri.extracted/_240.extracted]
└─$ strings zld_fsextract | head -200
__exec__: 
17:03:20
Build: date:%s, time:%s
Jun  9 2010
support version:
usage: %s <input zip file> {-t | <unzip path> {-s <SET_EXTRACTION_COMMAND> | [-f file1 [-f file2 ...] ] [-d dir1 [-d dir2 ...] ] [-i]} [-D destDir]} [-q]
%-20s:
%-20s:%s
name
nc_scope
scope
version
build_date
checksum
core_checksum
no set defined in this zipfile
COMPATIBLE_PRODUCT_MODEL[%d]=%s
no Compatible Product Model defined in this zipfile
%s/%s
%s%s
```

So what we know is that the **"zld_fsextract"** takes as {arg.1} the zip file -s {arg.2}  and -e {arg.3} {arg.4}  {arg.5} to perform the unzip.


These options are used by the "unzip" binary to unzip a file. Based on information found online and on a quick analysis, it seems that the binary calculates the unzip password in some way based on the binary name or the binary content. But we need to do further investigation on how it passes that argument.


A code sample with the argument indicators : 


<img src="{{ site.baseurl }}/assets/images/RE/Zyxel/ZY_PARA.png">

The fastest way to extract the files is to emulate the MIPS processor and execute the binary.


# Dynamic Analysis

Before we begin we need to understand on how we're supposed to emulate it, let's review the **"zld_fsextract"** to see the architecture that it's based on.


<img src="{{ site.baseurl }}/assets/images/RE/Zyxel/MIPS_ARCH.png">


The architecture is based on MIPS x32 therefore we need a flavor based on x32 let's proceed to download the necessary files to emulate the firmware.


  Rquired Files : 
```zsh
  ┌──(frey㉿frey)-[~/Desktop/Projects/Zyxel_310/Emulation]
└─$ wget https://people.debian.org/~aurel32/qemu/mips/vmlinux-3.2.0-4-5kc-malta ; \
    wget https://people.debian.org/~aurel32/qemu/mips/debian_wheezy_mips_standard.qcow2
```

And finally emulating the software : 

```zsh
┌──(frey㉿frey)-[~/Desktop/Projects/Zyxel_310/Emulation]
└─$ qemu-system-mips64 -M malta -kernel vmlinux-3.2.0-4-5kc-malta -hda debian_wheezy_mips_standard.qcow2 -append "root=/dev/sda1 console=tty0" -net nic -net user,hostfwd=tcp:127.0.0.1:2222-:22
 ```

And now we can upload inside of the Emulated space the **"zld_fsextract"**

```zsh
┌──(frey㉿frey)-[~/Desktop/Projects/Zyxel_310/Emulation]
└─$ scp -P 2222 zld_fsextract root@127.0.01:/tmp/
root@127.0.0.1's password: 
zld_fsextract                                                                                                                                              100%   92KB   6.6MB/s   00:00 
```

Executing the binary proves that the emulation works and the binary executes without a problem : 


<img src="{{ site.baseurl }}/assets/images/RE/Zyxel/ZY_EMU.png">


Now that we're set we can perform an one-liner mips emulator with strace to execute the binary parse the **"473AAAB0C0.bin"** and review all the syscalls that it made till the exit loop.

I made a directory called **"Emulation"** which consists of the following needed files : **"473AAAB0C0.bin"**, **"zld_fsextract"**, **"debian_wheezy_mips_standard.qcow2"**, **"vmlinux-3.2.0-4-5kc-malta"**

After seeing that the process worked correctly with a fully emulated environment, i decided  to try the single binary emulation provided by qemu. By using **"strace"** we can quickly see how the **"unzip"** binary was launched to find the ZIP password.

```zsh
sudo strace -f -s 199 qemu-mipsn32 ./zld_fsextract ./473AAAB0C0.bin  ./unzip -s extract -e code
```

<img src="{{ site.baseurl }}/assets/images/RE/Zyxel/ZY_ZIP_CREDS.png">


The binary that calculates the ZIP password is statically compiled, therefore it is not so easy to understand how that password is generated, and to be honest who cares?

# Extracting the Plaintext Firmware

Now that we have access to the plaintext filesystem, we can hook up the **"473AAAB0C0.bin"** inside Ghidra  using the password that we got from the strace syscalls and export the firmware locally.

Here's how Ghidra requests us for the password : 

<img src="{{ site.baseurl }}/assets/images/RE/Zyxel/GHIDRA_PW.png">

And that's the extracted version by using the password that we recovered from the encrypted binary :


<img src="{{ site.baseurl }}/assets/images/RE/Zyxel/GHIDRA_PLAIN.png">


And now we can export entirely the **"compress.img"** which is the plaintext firmware binary and analyse it locally for vulnerabilities.


<img src="{{ site.baseurl }}/assets/images/RE/Zyxel/my-job-here-is-done-bye.gif">

# Bonus


Seems like that the Zyxel hides some underlying users, but there's a check between **"pcVar27"** value and the **"iVar19"** which holds the account user of admin.

<img src="{{ site.baseurl }}/assets/images/RE/Zyxel/ZY_DEFAULT_PW.png">

The validation checks whenever the applied user is not in the UUID 0 which suggests that the admin account at **"pcVar27"** has full privileges over the system.




## Keynotes
{:.no_toc}

1. Encrypted firmwares can be reversed back to their plaintext version, you just need a brain for it
2. Static and Dynamic analysis play a huge role in the recovery process
3. Always search for string patterns in the binaries, they contain huge information for the underlying system


# Summary
{:.no_toc}

We've successfully performed  static and dynamic analysis to evaluate the flow chart, patterns and how the code works in general, decrypted on the fly the encrypted binary by abusing vendor applied tools listed on the binary itself. Finally we  exported the **"compress.img"** which was the plaintext binary all along. 