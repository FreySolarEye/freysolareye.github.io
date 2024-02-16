---
title: "A Tale of FRP Bypass - Lenovo"
date: "2024-02-03"
header:
  teaser: /assets/images/RE/IoT_Base.png
  teaser_home_page: true
tags: reverse,firmware
categories: RE
toc: true
toc_sticky: false
toc_label: "Table of Contents"
---
 
This blog elucidates an issue encountered during the initialization of a newly acquired Lenovo device, obtained through a contact in the Netherlands (thanks Alex), featuring Factory Reset Protection (FRP). My interest was piqued by the challenge posed in activating the device. The commencement of this investigative process involved an in-depth exploration of FRP, its inherent functionalities, and its potential to enforce a stringent lockdown on the device in instances of theft or to restrict access, even following a complete restoration to factory settings.
 
# Disclaimer 
{:.no_toc}

This is strictly for educational purposes <b>ONLY</b> and not to be used for conducting any illegal activities. I hold no responsibility for misuse of this information. 

# What is FRP

Factory Reset Protection (FRP) is a security feature implemented in mobile devices, primarily smartphones and tablets, to enhance the protection of personal data and prevent unauthorized access in case the device is lost, stolen, or reset to its factory settings. FRP is commonly associated with Android devices.

When FRP is enabled, it requires the user to enter the Google account credentials (username and password) that were previously associated with the device. This authentication is necessary after performing a factory reset or when setting up the device for the first time. The aim is to ensure that only the device owner, or someone with the authorized Google account information, can access and use the device.

FRP functions as a deterrent against theft and unauthorized use by discouraging potential intruders from resetting the device to gain access to it. It adds an additional layer of security beyond traditional screen lock methods. This feature is particularly useful in protecting sensitive information and maintaining the privacy of the device owner.

# Requirements
{:.no_toc}

  * Lenovo Tablet series (TB-X-MODEL)
  * Software Version TB-X505L-RF01_190519 or prior  




# Proof Of Concept
{:.no_toc}


Waiting for Lenovo Development Team approval, REF Ticket : LEN-150690

# The outcome

The outcome was complete unrestricted access to the device itself; Thus, bypassing the FRP and unlocking it completely.


How it began: 

<img src="/assets/images/IoT/FRP_Locked_Device.jpg">


How it ended:

<img src="/assets/images/IoT/Unlocked_Device.jpg">



# Timeline of disclosure to vendor

<b>04/01/2024</b>: The vendor was formally contacted, presenting a comprehensive Proof of Concept (PoC) alongside a demonstration video elucidating the FRP Bypass.

<b>05/01/2024</b>: Additional intricacies pertaining to the attack surface were furnished to the vendor, encompassing specific details such as the Android version and software version for a more thorough understanding.

<b>05/01/2024</b>: Subsequent to the provision of information, the vendor was contacted again and duly acknowledged the identified vulnerability. They proceeded to initiate communication with the development team at Lenovo.

<b>17/01/2024</b>: A follow-up communication was established with the vendor seeking an update concerning the FRP (Factory Reset Protection).

<b>17/01/2024</b>: The vendor responded, stipulating that the issue has been registered and is being tracked under the identifier LEN-150690.

<b>03/02/2024</b>: In response to a subsequent inquiry, the vendor communicated the current status of the vulnerability. Furthermore, it was conveyed that the software version in question has reached its End of Life (EOL) and consequently, no longer supports updates. The vendor explicated that tablets presently under active support do not permit the sharing of text before the End User License Agreement (EULA) is accepted, effectively mitigating the potential attack surface for FRP bypass on these devices.

<b>09/02/2024</b>: The vendor was contacted to give permissions for a public blog post, waiting  for approval.


# CVE Assign

Status : Waiting for CVE ID
