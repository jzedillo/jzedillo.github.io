---
title: "Bind Shell w/ remote command execution"
date: 2026-01-26
categories: malware analysis
tags: [malware, reversing, flarevm, remnux]
layout: single
author_profile: true
---

## Summary
In this blog I will be going over a malware sample that I pulled from HuskyHacks Practical Malware Analysis labs on Github, [[https://github.com/HuskyHacks/PMAT-labs](https://github.com/HuskyHacks/PMAT-labs)]. The malware sample being analyzed is named 'RAT.Unknown.exe.malz' however, do not let the name fool you as it is not a clear indicator of what the malware is. I will be going over some static and dynamic analysis techniques to discover host-based and network-based indicators. I hope you enjoy!

## Setup
- FlareVM
  - Procmon, TCPView, PE-bear/PEView, 
- REMnux
  - InetSim + Wireshark

## Static Analysis
First thing I always do before detonating malware is performing static analysis as it can help me discover IOCs at an early stage and help guide my analysis later on.

**Pull file hashes:**

`sha256sum RAT.Unknown.exe.malz`

`md5sum RAT.Unknown.exe.malz`

`sha1sum RAT.Unknown.exe.malz`

Next we want to automatically extract and deobfuscate all strings from the malware binary, using a tool known as *Floss* which is part of Mandiant's FLareVM toolset.
- Open cmder (or terminal of choice)
- Run `floss RAT.Unknown.exe.malz > floss.txt` 

The output is usually long and can be overwhelming, so dont get too stumped trying to find every single detail / IOC. Additionally not all IOCs will be found from here this is just helpful for early stage IOC findings that can help you narrow down your searching later on. 
{: .notice}

Scrolling through, the first thing that caught my eye was a particular Windows API function `InternetOpenUrlW`.

picture

This function is used to open a resource specified by a complete FTP or HTTP URL.

## Dynamic Analysis








