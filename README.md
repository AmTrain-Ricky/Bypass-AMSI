# Pwn-Town
Here are all of my notes for the Pwn Town lab!

Starting out
---
The first thing you want to do is get everything prepared. Some things I would to (to make your life easier) is:
* On your local computer, save the RDP configuration to make connecting easier
* Find the `pentest-tools` folder on the lab machine and pin that to quick access

Bypassing AMSI
---
The first major hurdle is bypassing AMSI, or Antimalware Scan Interface. After trying to run some of the powershell tools, you'll quickly notice they can't run because they "contain malware".

```diff
Import-Module .\PowerSploit.psd1
-This script contains malicious content and has been blocked by your antivirus software.
```

The way we can bypass this is a little weird (but works). We're going to compile a C file into a DLL, then load that into powershell. This part of bypassing AMSI was found on [this website](https://www.citadel.co.il/Home/Blog/1008) which took a long time to find!

If you want, you can just download the DLL from this repository and skip to [here](/#load-dll)

Now, you can [load the DLL into powershell](#load-dll)
