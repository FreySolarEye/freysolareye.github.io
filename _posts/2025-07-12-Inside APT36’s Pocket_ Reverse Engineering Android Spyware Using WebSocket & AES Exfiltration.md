---
title: "Inside APT36’s Pocket: Reverse Engineering Android Spyware Using WebSocket & AES Exfiltrations"
date: "2025-07-20"
header:
  teaser: /assets/images/malware/apt.jpg
  teaser_home_page: true
tags: reversing,malware
categories: reversing,malware
toc: true
toc_sticky: false
toc_label: "Table of Contents"
---
 
In this technical research paper, we dissect a mobile spyware strain targeting Android devices, discovered within the package namespace `io.bramerz.kfc`. Our investigation reveals connections to `APT36`, a threat actor historically targeting high-profile individuals and government entities.

---

# About the Authors

This research was conducted by **Frey** and **parad0x**, two independent security researchers with a shared interest in malware analysis and reverse engineering. 

- **Frey** works professionally in penetration testing, with a background in incident response and threat intelligence. Outside of work, he focuses on vulnerability research, malware analysis, and reverse engineering as a hobby.

- **parad0x** is a dedicated reverse engineer and malware researcher with a strong focus on Android malware. Specializing in software code analysis and internal weird code patterns.


## 1. Research Modus-Operandi?

We were both eager to dive into malware analysis, but for a while, we couldn’t find anything that really stood out. We kept digging, hoping to come across something unique  and eventually, we did kind of? The moment we found that particular malware sample, it just clicked. That’s when we decided to reverse it, and figure out exactly how it works.


## 2. Executive Summary

The malware demonstrates persistent background execution, data exfiltration via encrypted WebSocket communication `(Socket.IO)`, and improper use of `cryptographic` techniques.

**Key capabilities of the malware include:**

**Persistent Behavior:**
The malware ensures survivability across device reboots by leveraging broadcast receivers and foreground services. It monitors boot events and secret dial codes to stealthily restart its malicious core service (`MainService`).

**Modular Data Theft:**
The application modularly harvests SMS messages (`MsgMan.java`), contact data (`KontactsMan.java`), and local files (`FileMan.java`). Each module operates semi-independently, funneling data into a central exfiltration pipeline.

**Dual Exfiltration Channels:**
Stolen data is transmitted either via:

- Encrypted WebSocket (`IOSOK.java`) connections to hardcoded IP addresses using custom device metadata (e.g., UUID, model)

- HTTP POST uploads through static endpoints

**Weak Cryptography:**
The malware implements AES-128-CBC encryption with hardcoded keys and initialization vectors, rendering it trivially decryptable during analysis.

**Command and Control (C2):**
It connects to a known malicious server (`173.208.142.79:5128`) and additional RESTful services hosted under `myfiles.homes`, using encoded device attributes for identification.

**Excessive Permissions Abuse:**
The app requests and utilizes highly sensitive Android permissions, including:

- `READ_SMS`, `READ_CONTACTS`, `RECEIVE_BOOT_COMPLETED`, `REQUEST_INSTALL_PACKAGES`, and full file system access.
These permissions enable the malware to operate autonomously with minimal user interaction or suspicion.

**Detection Potential:**
Numerous static and behavioral indicators are present, including hardcoded keys, predictable service names, unencrypted socket initialization, and static infrastructure IPs all offering viable detection vectors for defenders.

## 3. Component Overview

The malware is composed of several key files:

| Component              | Description                                                                 |
|------------------------|-----------------------------------------------------------------------------|
| **Permissions Overview** | Extensive permissions that the malware requires                            |
| **MainActivity.java**   | Launches persistent background services                                     |
| **MyReceiver.java**     | Broadcast receiver for boot/startup persistence                             |
| **MainService.java**    | Foreground or background service to maintain persistent activity            |
| **Ad.java**             | Responsible for harvesting sensitive user data                              |
| **MsgMan.java**         | Handles SMS-related interception or exfiltration                            |
| **KontactsMan.java**    | Manages and collects contact data from the device                           |
| **FileMan.java**        | Manages file system exploration and data exfiltration                       |
| **HttpUploaderAll.java**| HTTP interface for uploading all collected data                             |
| **IOSOK.java**          | Initializes Socket.IO and generates a persistent UUID for tracking          |
| **URs.java**            | Hardcoded C2 infrastructure and endpoint mappings                           |
| **Encrypt.java**        | AES-128-CBC encryption using a fixed key and IV                             |


## 4. High-Level Execution Flow (Lifecycle Analysis)

**Phase 1: Startup & Persistence - "The Silent Infiltration**

1. Installation Trigger

The app icon looks harmless enough...
- `MainActivity` launches briefly - just long enough to trigger background services

2. Persistence Mechanism Trigger

Now we need to survive reboots...

- MyReceiver registers for: `BOOT_COMPLETED` (to survive restarts)

[![](https://mermaid.ink/img/pako:eNp9Vv1y2jgQfxWN-0fIDDEYQgO-m844thPc4EAxSdM2NxnFEuDDlnySSEOTzNyz3KPdk9xKhI8Wcv7DtrS7v_1e6clKOaGWa00ELqdoFNwyBI_3zRtcoIhJhfMcq4yzP1zXvc9xOntlQEdHH9DptxhnzEtV9pCphYs48wXFiv7CfGqY_W-3VkIV8nIsihgzPKECVZy6RBlTVDzgXB7-fi8-bNNtSdWQloCZsUnl8Nb6Bdk3yIFGVlgopM1JACtLqYYaUEZAMAJ8puwJVa-0PUhLG0NAGlBRZFKCz8if0nSmgYahF9wlcVJFvheHQ6-KklF_6J2HOzihxnmONQCbPKMzABxSkgmaKqQ4ggBoV6QGxYwInhHto9mzvcGgF_neKOpf3gXhyIt6yV0SjkbR5XnyhqJzgcE58ozO1zE4y3KKkhQzpuPb2IlvF_TmENqSSxXQHC8o2ROPwMSjC6hbMdUZNkp8XhQAsyPVNVIRSPkc9C999hvoM71PeDqjShsQ9W1pFhXnpGk36m3bOW7YJx235TTau5ZEBvOjqR5GUEC1JeAlg_ooBfjmatB___4Hxf0g7LnodJ7lxDaLNcW7vDrz_NHVMByuGbb2VnzeZTDsR8GK5TocJpAMexj2Qi8JV1yaIZliQclA0DEVlKVUVg7mLPtrTu8yclfCtjzYdeWjceXChMcEEPU4LwHMhMPmrHLABaFij-jyfWEAek8r8dGipC8_M_ZMXdQfvatnFIMmjxQZQyMsZy7yiI3n-7K9EjqLntElCJka6pdUmN6XLjqLKocr9-G_Xi2xmh66qJdJhcbALbeozooa8O8s55i8qS-Jn1FfZzZOEFSMEjyHaMQbXfBfB6BEUZzDItnad6qKVws50WRdGEB9U08Mfg1MK6YQYBRnqeDllDOo6DiqSJoahdc8S2Hw2FKX-JJ1_9hZvmOTjE86m2bqoW5GCGUoMN3OxUJj6kjaxQwmgKzU7GQhFS1qu4ifDNZQjx8InVyXdE2SFAuyllzvE6zwencHbmjgko1pPT4xg2GDnPOJrR6Vi4b4u8kgyiGXW9RXMkjB9KDkJ_J8LX1V6gQjJUA3FTuGJMaQERjSxeKBQrEEYPjaiAvIOU6V1FGH8ewvl_B52JRALCev5KSQe5KxVHGlqwiG3paXUO0gpCvU7FVMUUMidiGuDMQ1QIywABkUPsKJIU3payS7JOMqsglPH-HzmEv9-bOcwLsom2_Afd7ArSoi24p_bdUcm5wGfhTXfFxA2202P0-xkl5Z1mI4Rrb2A57OCzjW5G72r43-G11MIgOtKvtBf44LFnBuTKjOxZ6I3hj5LyAfslQsSoW-Z2qKvDA5ggmtIS4onPZ1p9E8br0_aXfwfUro2Mz2axeNKUnvcad98r513Gw49R38Lwb_K-C_Vs9DhlF3NBqY9u4qVS73qfDy3J6b_zdM_WqgPM_YSkoOx8EmxNObm1K6tVqxMBPKnvKCylqyYOlrocn_51xqvs4I5TGVEi4kO_q95UXIOzXOEN1qq94wFxBBHyBHSNCjJdgboyTNsZQBHSND0e2Yu-_q5qmmPOfCfTc2T1XCjJzR1fI3qwo3t4xYrhJzWrWgdAqsl9aTBr611JQWYLULvwSL2a11y15ApsTsK-fFSkzw-WRquWO4IsBqbvwIMgx3wg0LjFgqfD5nynKdjtMyIJb7ZD1abrtld5xmvdloNeFIbzfaVWthuUdOvdOxW61mq91uvz9x2ietl6r1w-ht2M1mu9msN5yW02l2Gs5x1YL6hh6Jl_dRcy19-Q_kjHE1?type=png)](https://mermaid.live/edit#pako:eNp9Vv1y2jgQfxWN-0fIDDEYQgO-m844thPc4EAxSdM2NxnFEuDDlnySSEOTzNyz3KPdk9xKhI8Wcv7DtrS7v_1e6clKOaGWa00ELqdoFNwyBI_3zRtcoIhJhfMcq4yzP1zXvc9xOntlQEdHH9DptxhnzEtV9pCphYs48wXFiv7CfGqY_W-3VkIV8nIsihgzPKECVZy6RBlTVDzgXB7-fi8-bNNtSdWQloCZsUnl8Nb6Bdk3yIFGVlgopM1JACtLqYYaUEZAMAJ8puwJVa-0PUhLG0NAGlBRZFKCz8if0nSmgYahF9wlcVJFvheHQ6-KklF_6J2HOzihxnmONQCbPKMzABxSkgmaKqQ4ggBoV6QGxYwInhHto9mzvcGgF_neKOpf3gXhyIt6yV0SjkbR5XnyhqJzgcE58ozO1zE4y3KKkhQzpuPb2IlvF_TmENqSSxXQHC8o2ROPwMSjC6hbMdUZNkp8XhQAsyPVNVIRSPkc9C999hvoM71PeDqjShsQ9W1pFhXnpGk36m3bOW7YJx235TTau5ZEBvOjqR5GUEC1JeAlg_ooBfjmatB___4Hxf0g7LnodJ7lxDaLNcW7vDrz_NHVMByuGbb2VnzeZTDsR8GK5TocJpAMexj2Qi8JV1yaIZliQclA0DEVlKVUVg7mLPtrTu8yclfCtjzYdeWjceXChMcEEPU4LwHMhMPmrHLABaFij-jyfWEAek8r8dGipC8_M_ZMXdQfvatnFIMmjxQZQyMsZy7yiI3n-7K9EjqLntElCJka6pdUmN6XLjqLKocr9-G_Xi2xmh66qJdJhcbALbeozooa8O8s55i8qS-Jn1FfZzZOEFSMEjyHaMQbXfBfB6BEUZzDItnad6qKVws50WRdGEB9U08Mfg1MK6YQYBRnqeDllDOo6DiqSJoahdc8S2Hw2FKX-JJ1_9hZvmOTjE86m2bqoW5GCGUoMN3OxUJj6kjaxQwmgKzU7GQhFS1qu4ifDNZQjx8InVyXdE2SFAuyllzvE6zwencHbmjgko1pPT4xg2GDnPOJrR6Vi4b4u8kgyiGXW9RXMkjB9KDkJ_J8LX1V6gQjJUA3FTuGJMaQERjSxeKBQrEEYPjaiAvIOU6V1FGH8ewvl_B52JRALCev5KSQe5KxVHGlqwiG3paXUO0gpCvU7FVMUUMidiGuDMQ1QIywABkUPsKJIU3payS7JOMqsglPH-HzmEv9-bOcwLsom2_Afd7ArSoi24p_bdUcm5wGfhTXfFxA2202P0-xkl5Z1mI4Rrb2A57OCzjW5G72r43-G11MIgOtKvtBf44LFnBuTKjOxZ6I3hj5LyAfslQsSoW-Z2qKvDA5ggmtIS4onPZ1p9E8br0_aXfwfUro2Mz2axeNKUnvcad98r513Gw49R38Lwb_K-C_Vs9DhlF3NBqY9u4qVS73qfDy3J6b_zdM_WqgPM_YSkoOx8EmxNObm1K6tVqxMBPKnvKCylqyYOlrocn_51xqvs4I5TGVEi4kO_q95UXIOzXOEN1qq94wFxBBHyBHSNCjJdgboyTNsZQBHSND0e2Yu-_q5qmmPOfCfTc2T1XCjJzR1fI3qwo3t4xYrhJzWrWgdAqsl9aTBr611JQWYLULvwSL2a11y15ApsTsK-fFSkzw-WRquWO4IsBqbvwIMgx3wg0LjFgqfD5nynKdjtMyIJb7ZD1abrtld5xmvdloNeFIbzfaVWthuUdOvdOxW61mq91uvz9x2ietl6r1w-ht2M1mu9msN5yW02l2Gs5x1YL6hh6Jl_dRcy19-Q_kjHE1)

<p align="center"><strong>Figure 1.</strong> Execution Flow of the Android Malware</p>

## 5. Core Components and Functional Behavior Analysis

### 5.1 Dangerous Permissions AndroidManifest.xml

The malware requests a comprehensive set of high-risk permissions in its `AndroidManifest.xml`. These permissions grant it unrestricted access to user data, hardware capabilities, and persistent background execution.


<p align="center">
  <img src="/assets/images/malware/figure2.png" alt="AndroidManifest.xml File" width="1000"/>
</p>

<p align="center"><strong>Figure 2.</strong> AndroidManifest.xml File</p>


### 5.2 MainActivity.java

Let’s break down the malware’s startup sequence together, notice how it balances stealth with functionality.

<p align="center">
  <img src="/assets/images/malware/figure3.png" alt="Execution Flow Diagram" width="1000"/>
</p>

<p align="center"><strong>Figure 3.</strong> Code Snippet: MainActivity Initialization</p>

**Trigger point - The Hook**

- The snippet executes in `MainActivity.onCreate()`, meaning the malware wakes up the moment the app launches.
- Why here? Android guarantees this runs, making it a reliable entry point. Clever, but not unusual.

**Service Orchestration - The Silent Backbone**

- The constructed `Intent` starts `MainService`, but here’s the twist:
- Persistence Module: Initializes immediately, pinging every `10` seconds (aggressive for a `"legit"` app).
- Delay Tactics: The service doesn’t stop the activity `(finish()` is intentionally missing), keeping the UI minimal to avoid suspicion.
- Thought process: "If we don’t close the activity, the app looks inactive, but the service keeps running in the background."

**Context Matters - Avoiding Pitfalls**

- This passes the Activity’s context to `startService()`.
- Developer’s oversight? Not quite. They’re careful, this context is valid only during startup, reducing memory leaks. A trade-off between reliability and stealth.

See the pattern yet? Every move screams persistence not speed. That’s how you stay hidden. Now let’s peel back the curtain and see what actually is really up to... 🙆‍♂️


### 5.3 MyReceiver.java

A `BroadcastReceiver` designed to maintain malware persistence by responding to system-wide broadcasts, such as `BOOT_COMPLETED` and custom secret codes.

<p align="center">
  <img src="/assets/images/malware/figure4.png" alt="BroadcastReceiver Initialization" width="1000"/>
</p>

<p align="center"><strong>Figure 4.</strong> Code Snippet: BroadcastReceiver Initialization</p>


From our view, this class acts like a secret switch. The dev (or malware author) can quietly trigger hidden features by dialing special numbers. It doesn’t ask for permission, doesn’t warn the user, and always runs a background service behind the scenes. Combined with the right payload, this is a classic `covert entry point` built to blend in, but capable of doing a lot under the radar.

- Listens for the special broadcast: `android.provider.Telephony.SECRET_CODE`.

- Parses the dialed secret code from the URI.

- Performs `different` actions based on the code:

- 8088: Opens `Notification Listener Settings`.

- 5055: Opens `App Details Settings `for androidx.multidex.

- `Always starts` MainService after receiving the broadcast.

### 5.4 MainService.java


Acts as the malware's **persistent execution core**, running indefinitely and orchestrating all data harvesting and communication modules. It ensures that once triggered (via `MainActivity` or `MyReceiver`), the malware sustains background activity while activating the full attack pipeline.

<p align="center">
  <img src="/assets/images/malware/figure5.png" alt="Snippet: IOSOK Initialization" width="1000"/>
</p>

<p align="center"><strong>Figure 5.</strong> Code Snippet: IOSOK Initialization</p>

<p align="center">
  <img src="/assets/images/malware/figure6.png" alt="KontactsMan Initialization" width="1000"/>
</p>

<p align="center"><strong>Figure 6.</strong> Code Snippet: KontactsMan Initialization</p>

<p align="center">
  <img src="/assets/images/malware/figure7.png" alt="Code Snippet: MsgMan Initialization" width="1000"/>
</p>

<p align="center"><strong>Figure 7.</strong> Code Snippet: MsgMan Initialization</p>

<p align="center">
  <img src="/assets/images/malware/figure8.png" alt="Code Snippet: FileMan Initialization" width="1000"/>
</p>

<p align="center"><strong>Figure 8.</strong> Code Snippet: FileMan Initialization</p>

Looking at MainService, it’s obvious this class is the heart of the malware. It runs as a background Android service that connects to a remote `Socket.IO` server and listens for commands. These commands enable a wide range of malicious behaviors, such as reading SMS, accessing contacts, stealing files, sending SMS, recording audio, scanning Wi-Fi networks, checking app permissions, and more.

It's designed to `persist`, hide itself using a `foreground service`, and communicate silently with a `C2 server`. This is a "classic" spyware with modular, stealthy, and persistent capabilities. From here on out, we can say with 100% confidence and maybe a little dramatic flair that this thing is definitely up to no good. Yep, it’s malware, no doubt about it like we didn't really see more suspicious activities up ahead....

- `MainService` extens `service` and overrides `onStartCommand()`.
- Uses `Handler.postDelayed()` loop, which ensures `constant background execution`.
- Sets up a **Socket.IO listener** in `"order"` to receive remote commands.

<p align="center">
  <img src="/assets/images/malware/houston.png" alt="Uhhh it's a malware" width="1000"/>
</p>


### 5.5 Ad.java

It harvests sensitive user data, stores it in hidden directories, and uploads it to a remote server

<p align="center">
  <img src="/assets/images/malware/figure9.png" alt="Code Snippet: Initialization of hidden directories and log files" width="1000"/>
</p>

<p align="center"><strong>Figure 9.</strong> Code Snippet: Initialization of hidden directories and log files.</p>

<p align="center">
  <img src="/assets/images/malware/figure10.png" alt="Code Snippet: Scans for multiple directories" width="1000"/>
</p>

<p align="center"><strong>Figure 10.</strong> Code Snippet: Scans for multiple directories</p>

<p align="center">
  <img src="/assets/images/malware/figure11.png" alt="Code Snippet: Usage of HttpUploaderAll.uploadData" width="1000"/>
</p>

<p align="center"><strong>Figure 11.</strong> Code Snippet: Usage of HttpUploaderAll.uploadData</p>

<p align="center">
  <img src="/assets/images/malware/figure12.png" alt="Code Snippet: Usage of getListFiles" width="1000"/>
</p>

<p align="center"><strong>Figure 12.</strong> Code Snippet: Usage of getListFiles</p>

<p align="center">
  <img src="/assets/images/malware/figure13.png" alt="Code Snippet: Usage of arrangeData" width="1000"/>
</p>

<p align="center"><strong>Figure 13.</strong> Code Snippet: Usage of arrangeData</p>

<p align="center">
  <img src="/assets/images/malware/figure14.png" alt="Code Snippet: Usage of uploadFiles" width="1000"/>
</p>

<p align="center"><strong>Figure 14.</strong> Code Snippet: Usage of uploadFiles</p>

The Ad class appears to be a backend utility within the app responsible for gathering and managing a large variety of potentially sensitive files from the device's storage. It scans common directories and filters for `document`, `spreadsheet presentation`, and `image` file types, logging their paths and then uploading these files to a remote location via `HTTP` requests.

The class keeps track of which files have been processed and uploaded through local log files to avoid `duplication`. It also interacts with other parts of the malware to extract contacts and SMS data before proceeding with file uploads.


- Use of hidden directories like `/.System/` to store logs and data suggests intent to hide activity.

- Collection of wide file types including **documents**, **images**, and **spreadsheets** which may contain `private info`.

- Frequent uploading of files without user awareness.

- Interaction with `contact` and `SMS managers` for data collection.

- Overwriting and managing logs to track data already sent, indicating stealth and persistence.


### 5.6 MsgMan.java

It harvests SMS data and stores or potentially exfiltrates them.

<p align="center">
  <img src="/assets/images/malware/figure15.png" alt="Code Snippet: Usage of getsms() function inside MsgMan" width="1000"/>
</p>

<p align="center"><strong>Figure 15.</strong> Code Snippet: Usage of sendSMS() function inside MsgMan</p>


<p align="center">
  <img src="/assets/images/malware/figure16.png" alt="Code Snippet: Usage of getsms() function inside MsgMan" width="1000"/>
</p>

<p align="center"><strong>Figure 16.</strong> Code Snippet: Usage of sendSMS() function inside MsgMan</p>

It is designed to quietly extract all SMS messages from an Android device, convert them into `JSON` or `CSV` formats, and store them inside hidden directories. It accesses the SMS inbox, reads messages content and metadata, and writes them into a CSV file for later retrieval or upload.

Additionally, it provides a method to send SMS messages silently, without user confirmation or interface, allowing remote control over outgoing messages.

In short, this class acts as a stealth SMS data collector and message sender, commonly seen in spyware or malware to both exfiltrate sensitive communication data and possibly? Send messages on behalf of the user for malicious purposes.

- Silent extraction of all SMS messages into hidden storage.

- Exporting sensitive SMS data (addresses and bodies) without user knowledge.

- Silent SMS sending capability, enabling potential misuse such as spam or command/control messages.

- Use of hidden directories (e.g., `/.System/sm.csv/`).

- No user interaction or permission checks visible in the class (likely relying on pre-granted permissions).


### 5.7 KontactsMan.java

It accesses all contacts silently using a background context from `MainService` which is designed for easy json serialization and data exfiltration.

<p align="center">
  <img src="/assets/images/malware/figure17.png" alt="Code Snippet: Usage of getContacts function inside KontactsMan" width="1000"/>
</p>

<p align="center"><strong>Figure 17.</strong> Code Snippet: Usage of getContacts function inside KontactsMan</p>

<p align="center">
  <img src="/assets/images/malware/figure18.png" alt="Code Snippet: Usage of getContactsForApp function inside KontactsMan" width="1000"/>
</p>

<p align="center"><strong>Figure 18.</strong> Code Snippet: Usage of getContactsForApp function inside KontactsMan</p>

<p align="center">
  <img src="/assets/images/malware/figure19.png" alt="Code Snippet: Usage of getContactConv() function inside KontactsMan" width="1000"/>
</p>

<p align="center"><strong>Figure 19.</strong> Code Snippet: Usage of getContactConv() function inside KontactsMan</p>

The KontactsMan class accesses the phone's contact list and can output it in two main formats: `JSON` for internal processing or `CSV` stored in a hidden directory. This means it can gather full contact details **(names and phone numbers)** from the device without user interaction and store or send them as needed.

The class allows both JSON retrieval for use within the app and file export for persistent storage or later exfiltration. The contacts’ phone numbers are cleaned to remove spaces and dashes, making the data easier to process downstream.

- Contacts are collected silently without any user prompt or UI feedback.

- Data is written to hidden system-like folders (`/.System/Ct.csv`), suggesting attempts to conceal the files.

- Integration with a background service context suggests this happens without explicit user awareness.

- Exporting contacts to CSV files facilitates easy data extraction or upload.

### 5.8 FilemanMan.java

Scans and lists all files & folders in the given directory path and uploads the specified files.

<p align="center">
  <img src="/assets/images/malware/figure20.png" alt="Code Snippet: Usage of Fileman class" width="1000"/>
</p>

<p align="center"><strong>Figure 20.</strong> Code Snippet: Usage of Fileman class</p>

<p align="center">
  <img src="/assets/images/malware/figure21.png" alt="Code Snippet: Usage of downlFile function inside of Fileman" width="1000"/>
</p>

<p align="center"><strong>Figure 21.</strong> Code Snippet: Usage of downlFile function inside of Fileman</p>

Acts as a file browser and uploader on the device. It can list the contents of any given directory (`except hidden files/folders)`, reporting back as JSON. If access is denied, it notifies a remote server through a socket connection with an error message.

- It also allows uploading any file given its path, sending it through an HTTP uploader utility tied to the app’s background service context.

- Silent browsing of arbitrary file system paths.

- Error notifications sent via socket to a remote listener `(IOSOK.getInstance().getIoSocket().emit)`.

- File uploads without user consent or interaction.

- Filtering out hidden files/directories, possibly to avoid exposing system or app-internal files.

### 5.9 HttpUploaderAll.java

It exfiltrates arbitrary files from the Android device to a remote-controlled server.

<p align="center">
  <img src="/assets/images/malware/figure22.png" alt="Code Snippet: Usage of uploadData function inside of HttpUploaderAll" width="1000"/>
</p>

<p align="center"><strong>Figure 22.</strong> Code Snippet: Usage of uploadData function inside of HttpUploaderAll</p>

<p align="center">
  <img src="/assets/images/malware/figure23.png" alt="Code Snippet: Usage of getMimeType function inside of HttpUploaderAll" width="1000"/>
</p>

<p align="center"><strong>Figure 23.</strong> Code Snippet: Usage of getMimeType function inside of HttpUploaderAll</p>

 It handles uploading files from the infected device to a remote server controlled by the attacker. It sends files in multipart form along with a `unique device ID` for tracking. The class smartly detects the file type based on extension, so the server receives accurate `content-type` information.

Uploads happen silently and asynchronously, with logs only on success or failure for debugging. The destination URL includes a **fixed IP** and a **specific port**, hinting at a dedicated backend waiting for incoming stolen data. 

- Uploads arbitrary files without user consent.

- Uses a unique device identifier to track victims individually.

- Connects to a suspicious custom endpoint with a non-standard port.

- Transmits potentially sensitive user data (`images, documents, APKs`) to an unknown remote server.

- The method is static and can be called from anywhere in the app (as seen in `FileMan`), facilitating data exfiltration on demand.

### 5.9 IOSOK.java

Main communication channel to attacker-controlled server via Socket.IO.

<p align="center">
  <img src="/assets/images/malware/figure24.png" alt="Code Snippet: Usage of IOSOK" width="1000"/>
</p>
<p align="center"><strong>Figure 24.</strong> Code Snippet: Usage of IOSOK </p>

Acts as the core networking and device identity manager for the malware. It ensures every infected device gets a persistent, unique identifier and establishes a persistent WebSocket connection to a remote command-and-control server.

By sending device-specific parameters `(model, manufacturer, OS version, and ID)`, the server can track and identify devices individually in real-time. The socket’s auto-reconnect configuration means the app maintains nearly continuous contact with the attacker’s backend.

- This class is crucial for maintaining a live communication channel, used to receive commands, send status updates, or stream stolen data.

- Persistent device identifier generation and storage, enabling long-term victim tracking.

- Continuous WebSocket connection to a suspicious server `IP` on a non-standard port `(5128)`.

- Transmitting sensitive device metadata without user consent.

Let’s not forget the reconnection delay maxing out at `999999999`. Someone really panicked when testing disconnections....🙆‍♂️


### 5.10 URs.java

Hardcoded C2 infrastructure and endpoint mappings.

<p align="center">
  <img src="/assets/images/malware/figure25.png" alt="Code Snippet: " width="1000"/>
</p>
<p align="center"><strong>Figure 25.</strong> Code Snippet: Usage of  URs </p>

 Centralizes all network endpoints and cryptographic constants used by the malware for communicating with its backend infrastructure. The backend domain `myfiles.homes` hosts multiple HTTP endpoints handling user accounts, messaging, contacts, and media uploads.

Hardcoded encryption keys suggest some payload or data `protection/encryption`, although fixed keys reduce security.

Separately, the IP address points to a different communication channel `(for sockets or file uploads)`, showing a split architecture between HTTP and raw socket/file communication.

- Stores hardcoded C2 IP `(173.208.142.79)` used for real-time socket communication.
- Contains full set of HTTP endpoints under `https://myfiles.homes.`
- Exposes static AES key and IV used for symmetric encryption operations.


### 5.11 Encrypt.java

Enctryption methods

<p align="center">
  <img src="/assets/images/malware/figure26.png" alt="Code Snippet: " width="1000"/>
</p>
<p align="center"><strong>Figure 26.</strong> Code Snippet: Usage of Encrypt </p>


<p align="center">
  <img src="/assets/images/malware/figure27.png" alt="Code Snippet: " width="1000"/>
</p>
<p align="center"><strong>Figure 27.</strong> Code Snippet: Usage of Encrypt </p>

It encrypts and decrypts strings using AES with fixed key and IV values from the `URs config`. The encryption is custom-padded using null characters, and data is typically hex-encoded when stored or transmitted.

Given these hardcoded parameters, any encrypted message can be decrypted easily if someone obtains this code or extracts it from the app

 Implements `AES` encryption in `CBC` mode with NoPadding using static key and IV.

- Manually applies `zero-byte` padding to align plaintext to `AES` blocks.

- Decrypts hex-encoded strings and strips trailing `0x00` padding bytes.

- Converts byte arrays to and from hexadecimal using custom helpers.

- Uses hardcoded `key/IV` from `URs`, making decryption reproducible during analysis.


## 6. Indicators of Compromise (IoCs)

This document contains a detailed list of IoCs based on the reverse engineering and behavioral analysis of the Android malware components.

---

### 6.1 Network Indicators (C2 Servers, Endpoints)

URLs / IPs / Endpoints

```
http://<URs.ip>:9108/hmm         // File upload endpoint
http://<URs.ip>:5128             // Socket.IO Command & Control (C2)
```
 Extract actual IP/domain from the `URs` class for final IOC listing.

---

### 6.2 App Package Information

App Package Name

```
io.bramerz.kfc.kaam
```

Modules include: `KontactsMan`, `MsgMan`, `HttpUploaderAll`, `IOSOK`, etc.

---

### 6.3 File System Artifacts

Storage Paths Created or Written

```
/storage/emulated/0/.System/sm.csv/       // SMS dump
/storage/emulated/0/.System/Ct.csv/       // Contacts dump
/data/.System/sm.csv/                     // Alternate SMS path
```

These hidden `.System/` directories are used for stealth.

---

### 6.4 Persistence Identifiers

SharedPreferences Key

```
SharedPreferences: "unique_id_prefs"
Key: "unique_id"
```

Used for device tracking across sessions.

---

### 6.5 Socket.IO Parameters (Victim Fingerprint)

Socket Connection Info:

```
http://<URs.ip>:5128?model=...&manf=...&release=...&id=<uuid>
```

Transmits:
- Device model
- Manufacturer
- Android version
- Unique UUID

---

### 6.6 Behavioral IoCs (Malicious Capabilities)

| Capability        | Module/Class          | Behavior |
|------------------|------------------------|----------|
| Read all SMS     | `MsgMan.getsms()`      | Reads all messages |
| Export SMS to CSV| `MsgMan.getSms()`      | Dumps inbox to file |
| Send SMS         | `MsgMan.sendSMS()`     | Sends SMS silently |
| Read Contacts    | `KontactsMan.getContacts()` | Queries contact list |
| Export Contacts  | `KontactsMan.getContactConv()` | Dumps to `.csv` |
| File Traversal   | `FileMan.walks()`      | Lists accessible files |
| File Upload      | `HttpUploaderAll.uploadData()` | Sends to C2 |
| Real-time C2     | `IOSOK.getIoSocket()`  | Persistent socket C2 |

---

### 6.7 Additional Technical Fingerprints

Libraries Used

- `com.opencsv.CSVWriter`
- `okhttp3.OkHttpClient`
- `io.socket.client.Socket.IO`
- `org.apache.commons.lang3.StringUtils`

   MIME Type Detection Map

- `jpg`, `pdf`, `docx`, `apk`, `mp4`, etc.

---

## 7.0 Summary of Key IoCs

| Type               | Value |
|--------------------|-------|
| Package Name       | `io.bramerz.kfc.kaam` |
| Storage Path       | `.System/sm.csv/`, `.System/Ct.csv/` |
| SharedPreferences  | `unique_id_prefs` → `unique_id` |
| C2 Server (Socket) | `http://<URs.ip>:5128` |
| C2 Server (HTTP)   | `http://<URs.ip>:9108/hmm` |
| Protocols          | Socket.IO, HTTP (multipart) |
| Libraries          | `okhttp3`, `socket.io-client`, `opencsv` |
| Behavioral         | SMS, Contacts, File Exfiltration |

## 8.0 Personal Notes

 This malware left us with a mix of respect and concern. And here’s what stood out technically, tactically, and sometimes tragically.

### 8.1 Websocket Exfiltration on android?

This isn’t your usual lazy HTTP POST to an obscure PHP backend. No, this spyware goes full-duplex, opening a persistent channel bi-directional and suspiciously  interested in fingerprinting things.

###  8.2 AES Encryption... With Hardcoded Keys and IVs

They went out of their way to include AES-CBC encryption. Respect.
Then they hardcoded the key and IV as plain strings in the `URs` class like it’s 2009 and nobody told them about memory forensics.

`SecretKey: 0123456789abcdef`
`IV: fedcba9876543210`

### 8.3 SECRET_CODE Intent Trigger

Ah, the `android.provider.Telephony.SECRET_CODE` intent. A clever and very Android-specific trick to trigger your malware without waking it up via icon taps or push messages.

Except... it only works on some manufacturers, and in 2025, hardly anyone’s building stealth payloads off dialer codes anymore. It’s like hiding your spyware behind a payphone

### 8.4 C2 Over Hardcoded IP?

Yes, they use WebSockets.
Yes, they use HTTPS endpoints with domains like `https://myfiles.homes` to blend in.

We had to double-check the code multiple times. This is nation-state malware, and they’re just throwing a raw IP in there...

### 8.5 The "myfiles.homes" C2 Naming Trick

Gotta hand it to them: `myfiles.homes` sounds totally benign. Like a Google Drive knockoff.

They leaned into deceptive simplicity and that’s smart. In a sea of weird malware domains, this one looks like a feature??

### 8.6 Using OkHTTP

You can tell some of these folks write consumer apps on the side. They used OkHttpClient, async calls, callbacks all the shiny toys.

This isn’t your usual malware written in barely-patched Android Studio. This is someone who’s published on the Play Store...

### 8.7 Encryption PAd Strategy: Down to the rabbit hole

Their padding logic in AES encryption just... slaps zeroes onto the string until it fits the block size.

Like literally:

```
str = str + (char) 0;
```

No PKCS#7, no cleverness. Just throw some zeroes at it and hope nobody notices.


## 9.0 Conclusion

This malware is a weird mix of competence and corner-cutting. It’s got clean architecture, some surprising stealth, and modern development practices. But it’s also riddled with OPSEC failures, questionable cryptography, and network decisions that practically beg for detection. From an analyst’s lens, that’s what makes it so interesting. Not the code itself but what the code reveals about the people behind it. Skilled, fast-moving, maybe under pressure, and not as invincible as they’d like you to think.