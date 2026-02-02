---
title: "Malware Dropper + Bind Shell"
date: 2026-01-26
categories: malware analysis
tags: [malware, reversing, flarevm, remnux]
layout: single
author_profile: true
---

## Summary
In this blog I will be going over a malware sample that I pulled from HuskyHacks [Practical Malware Analysis labs][Github] on Github. The malware sample being analyzed is named 'RAT.Unknown.exe.malz' however, do not let the name fool you as it is not a clear indicator of what the malware is. I will be going over some static and dynamic analysis techniques to discover host-based and network-based indicators. I hope you enjoy!

[Github]: https://github.com/HuskyHacks/PMAT-labs

**Setup**
- FlareVM
  - Procmon, TCPView, PE-bear/PEView 
- REMnux
  - InetSim + Wireshark

## Static Analysis
First thing I always do before detonating malware is performing static analysis as it can help me discover IOCs at an early stage and help guide my analysis later on.

**Pull file hashes:**
- `sha256sum RAT.Unknown.exe.malz`
- `md5sum RAT.Unknown.exe.malz`
- `sha1sum RAT.Unknown.exe.malz`

This is useful as we can take the file hashes and run them through VirusTotal, or other OSINT tools, to see if they have been seen in the wild before. 

**Extract all strings from the file:**

Next we want to automatically extract and deobfuscate all strings from the malware binary, using a tool known as *Floss* which is part of Mandiant's FLareVM toolset.
- Open cmder (or terminal of choice)
- Run `floss RAT.Unknown.exe.malz > floss.txt` 

**Note:** The output is usually long and can be overwhelming, so dont get too stumped trying to find every single detail / IOC. Additionally not all IOCs will be found from here this is just helpful for early stage IOC findings that can help you narrow down your searching later on. 
{: .notice--primary}

Scrolling through, the first thing that caught my eye was a particular Windows API function `InternetOpenUrlW`, which was followed by some intersting strings in the picture below.

![Floss Output](/assets/images/RAT1/floss.png)

This function is used to open a resource specified by a complete FTP or HTTP URL as referrenced here [MalApi][MalApi]. We can infer that this function is used to download a web resource from the observed domain, likely an additional payload. Furthermore, we see "*what command can I run for you*" which likely indicates there is some remote command execution capabalites, however we cannot fully confirm based on that alone, until we have fully analyzed this binary. 

[MalApi]: https://malapi.io/

## Dynamic Analysis
Now we can go ahead and start with detonating the file and seeing what kind of host-based and network-based indicators we observe. 

I like to detonate malware a couple times to see if we can observe any different behaviors or IOCs. For example if this lab had a real internet connection, maybe we would observe HTTP requests to multiple differnet domains, making it harder to catch. In this case we will only see request to my InetSim domain, however we can also check what happens if we detonate it with and without an "internet" connection.  

**1st Detonation:**
- Do not start up InetSim, and just detonate malware.

When we do this we observe a pop up message saying "*NO SOUP FOR YOU*” which is one of the early indicators we observed when extracting the obfuscated strings from the binary.

![No Soup For You](/assets/images/RAT1/NOSOUPFORYOU.png)

**Note:** Make sure you are restoring from a pre-detonation snapshot every time you want to perform another detonation, as we do not fully know what is being installed, written, changed, etc.
{: .notice--primary}

**2nd Detonation**
- Start up InetSim + Wireshark on our REMnux machine → detonate the file again.

Looking in wireshark, we observe a successful (200) ‘*GET*’ request to a specified server (our fake internet simulator) for a file named [msdcorelib.exe].

![Wireshark traffic](/assets/images/RAT1/wireshark.png)

Right now, we have no clue where that file is being downloaded to on our machine, could be a temp folder, downloads folder, who knows. If we try and search for that file in our file system, we do not see it. Therefore we will have to detonate it again and use different tools. 

**3rd Detonation:** 
- On FlareVM → open procmon → select fitler icon → filter on 'process contains RAT.Unknown.exe' and 'operation contains File' → detonate file again.

Scrolling through we will see a lot of *CreateFile* and *WriteFile* events, however there is a particular event that catches my eye. We see a CreateFile and WriteFile event located at the startup directory [C:\Users\sherlock\AppData\Roaming\Microsoft\Windows\Start Menu\ Program\Startup\mscordll.exe]. 

![ProcMon](/assets/images/RAT1/procmon.png)

If we keep looking through the file operations, we will not see the previous file we were looking for [msdcorelib.exe]. This is likely because when you download from a web resource, the data of the download can be transmitted first → then written to the file system to another name. That is likley what is happening here and is a common masquerading / evasion technique. 

We can verify the file is there by navigating to that file path

![Startup](/assets/images/RAT1/startup.png)

There it is. 

Unfortunaltey, in this setup we cannot see what this file does as this is just the InetSim default GUI binary. However this represent there was a download from the internet [msdcorelib.exe] and then written to the file system as [mscordll.exe] in the startup directory which is a common persistance / evasion technique. 

Lastly, we have one more tool left to check.

**4th Detonation:** 
 - On FlareVM → open TCPView  

We are using this tool as it will show us a real-time view of all active TCP and UDP endpoints. This is helpful because malware is often seen using TCP for reliable C2 communication.
{: .notice--primary}

Detonate the file again and scroll to find our malicious file.

We can see the file is listening on TCP port 5555.

![TCP View](/assets/images/RAT1/tcpview.png)

On REMnux → open terminal → attemp to connect to our FlareVM host using netcat `nc -nv 10.0.0.4 5555`

![Netcat Connection](/assets/images/RAT1/NCconnection.png)

WE ARE IN!!!

We are greeted with what appears to be a base64 encoded string, therefore we will decode it to see what it says.

Open another tab in your terminal → run `echo “base64 string” | base64 -d` and we see it says “what command can I run for you”. 

Now we can test out the extent of what commands we can run through here.

![netcat cmds](/assets/images/RAT1/netcat.png)

## Conclusion
After gathering all of our findings we can infer that this malware deploys a bind shell, that provides remote command execution capabilities and deploys an additional payload in the startup folder for persistence. 

This activity is similar to a RAT as they have persistence, command execution and file download capabilities however, the clear distinction is that it opens a TCP socket and listens for incoming connections. 

That's all folks, I hope you enjoyed!
