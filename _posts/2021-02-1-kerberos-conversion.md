---
layout: post
title: Experimenting with Kerberos
subtitle: Extracting Kerberos tickets with Rubues and utilizing them in various formats
thumbnail-img: /assets/posts/kerberos-conversion/thumb.png
share-img: /assets/posts/kerberos-conversion/thumb.png
tags: [kerberos, rubeus, impacket, .kirbi, .ccache]
---

On recent pentests, I've been attempting to leverage unconstrained Kerberos delegation and stolen Kerberos tickets more. This is one of those privilege escalation methods that can fly under the radar - you might not need it to get the job done, but under the right conditions it might exactly what you're looking for. One of the things I found most confusing when starting with stolen TGTs (ticket-granting-tickets) was the different formats you can prepare the tickets in for usage with various offensive tools. This post will cover how you can use tickets snatched with Rubeus with different tools like Impacket, CrackMapExec, Mimikatz and Rubeus itself.

*Note: If you're looking for Kerberos delegation background or attack scenarios, [this](http://www.harmj0y.net/blog/redteaming/not-a-security-boundary-breaking-forest-trusts/) is probably the most helpful post I found

# Stealing a Ticket
First off, we need to obtain a ticket before we can play around with formatting. In the test lab, I've got a Windows 10 system configured for unconstrained Kerberos delegation:

![Delegation setting](/assets/posts/kerberos-conversion/delegation-setting.png){: .mx-auto.d-block :}

Assuming that we've already compromised an account with local admin rights to the target, I'll use [Rubeus](https://github.com/GhostPack/Rubeus) to extract tickets from memory. This can be done with the `monitor` command:

~~~
Rubeus.exe monitor /interval:1
~~~

Now it may be sufficent to just wait and see what privileged or high-interest users/computers authenticate to our compromised host, or it may be possible to force a sensitive system to authenticate through the printerbug. In this case, I've just waiting until a domain admin, `vizimir`, has logged in and pulled their TGT:

![Vizimir TGT](/assets/posts/kerberos-conversion/capture-viz-tgt.png){: .mx-auto.d-block :}

Now that we've got a ticket, we can explore the various tools that can utilize them and the formats required.

# Rubeus (base64 or .kirbi)
We'll start with the easiest one. Rubeus can import a TGT to the current logon session from either a `base64` string or a `.kirbi` file. We'll stick with base64 since it's the most straightforward and also the format in which Rubeus initally presents the ticket to us.

Before we start, let's verify that our current logon session (as `REDANIA\dijkstra`) can't access C$ on the domain controller:

![DC Fail](/assets/posts/kerberos-conversion/rubeus/verify-no-dc-access.png){: .mx-auto.d-block :}

Great, so if we successfully import a Domain Admin's TGT to our session, we should then be able list folders on the DC.

This is pretty simple with Rubues - all we have to do is supply the base64 ticket to the `ptt` command:

~~~
Rubeus.exe ptt /ticket:<base64 blob>
~~~

![ptt](/assets/posts/kerberos-conversion/rubeus/rubeus-ptt.png){: .mx-auto.d-block :}

We can then use the `klist` command to verify the ticket has been imported to our session:

~~~
Rubeus.exe klist
~~~

![klist](/assets/posts/kerberos-conversion/rubeus/rubeus-klist.png){: .mx-auto.d-block :}

And there we can see the ticket for `vizimir` successfully imported to our session. Now if we try accessing C$ on the domain controller again, we see it's successful!

![ptt](/assets/posts/kerberos-conversion/rubeus/access-dc.png){: .mx-auto.d-block :}

This is a simple test of the new privileges, but we could also use tools like PsExec to remotely execute commands now.

One thing to note with utilizing Kerberos authentication is that Kerberos relies on domain names and DNS entries. If we rerun the `dir` command and target and IP address, NTLM authentication will be used and we'll get access denied still:

![IP Fail](/assets/posts/kerberos-conversion/rubeus/ip-fail.png){: .mx-auto.d-block :}

# Mimikatz (.kirbi)
Mimikatz can import Kerberos tickets to the current session in the form of `.kirbi` files. Before starting, we'll again verify we don't have rights on the domain controller:

![DC Fail](/assets/posts/kerberos-conversion/mimikatz/verify-no-dc-access.png){: .mx-auto.d-block :}

We can use a PowerShell oneliner to write the base64 ticket grabbed with Rubeus out to a file:

```powershell
[IO.File]::WriteAllBytes("C:\output.kirbi", [Convert]::FromBase64String("<base64 blob>"))
```

![Write Kirbi](/assets/posts/kerberos-conversion/mimikatz/write-kirbi.png){: .mx-auto.d-block :}

Boot up Mimikatz and import the ticket:

~~~
kerberos::ptt <path to kirbi file>
~~~

![Import Kirbi](/assets/posts/kerberos-conversion/mimikatz/import-kirbi.png){: .mx-auto.d-block :}

To test if it worked, we'll start a new command prompt with `misc::cmd` and verify we can access the DC:

![DC Access](/assets/posts/kerberos-conversion/mimikatz/access-dc.png){: .mx-auto.d-block :}

W00t! While we're here in Mimikatz let's just verify we can do a dcsync:

![DCSync](/assets/posts/kerberos-conversion/mimikatz/dcsync.png){: .mx-auto.d-block :}

# Impacket and CrackMapExec (.ccache)
This is by far my preferred method for using a stolen ticket, but it's also the one with the most formatting steps. We need to get the base64 ticket into a `.ccache` file. We can do this by converting from a `.kirbi` file, using Impacket's `ticketConverter.py`. 

First, format the base64 ticket to remove line breaks, spaces, etc. and then decode it with the `base64` command:

```bash
base64 -d <base64 ticket file> > <output .kirbi> 
```

![Convert to Kirbi](/assets/posts/kerberos-conversion/impacket/convert-to-kirbi.png){: .mx-auto.d-block :}


Convert to `.ccache` file using Impacket:

```bash
python3 ticketConverter.py <input kirbi file> <output ccache file>
```

![Convert to ccache](/assets/posts/kerberos-conversion/impacket/kirbi-to-ccache.png){: .mx-auto.d-block :}

Now that the ticket is in the right format, we're almost ready to use it. We just need to export the `KRB5CCNAME` variable and set it to the path of our `.ccache` file:

```bash
export KRB5CCNAME=/path/to/ticket.ccache
```

![Export KRB5CCNAME](/assets/posts/kerberos-conversion/impacket/export-krb5ccname.png){: .mx-auto.d-block :}

Setup complete! Impacket and CrackMapExec have a `-k` flag specifying to use Kerberos authentication. This will draw from the `KRB5CCNAME` variable we set. Let's verify we can access the DC:

![CME](/assets/posts/kerberos-conversion/impacket/cme-example.png){: .mx-auto.d-block :}

![Wmiexec](/assets/posts/kerberos-conversion/impacket/impacket-example.png){: .mx-auto.d-block :}

Again, be careful to use DNS names since IP will fail:

![IP Fail](/assets/posts/kerberos-conversion/impacket/ip-fail.png){: .mx-auto.d-block :}