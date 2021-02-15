---
layout: post
title: SOCKS Proxy Relaying
subtitle: Transitioning to multi-relay attacks with ntlmrelayx
thumbnail-img: /assets/posts/socks-relay/thumb.png
share-img: /assets/posts/socks-relay/thumb.png
tags: [ntlmrelayx, responder, relay]
---

I came across this [SecureAuth blog post](https://www.secureauth.com/blog/what-is-old-is-new-again-the-relay-attack/) recently and was amazed at some of the `ntlmrelayx.py` functionality I'd been missing out on. To over-simplify it, just throwing the `-socks` flag allows you to store sessions gained from authentication relays at a SOCKS proxy. You can then use other tools in conjunction with `proxychains` to take advantage of these stored sessions multiple times.

# Traditional Relay
Let's start with the way I've always run relay attacks in the past and a scenario in which utilzing the SOCKS feature becomes super beneficial.

As always, get Responder configured with `SMB` and `HTTP` off in `Responder.conf`:
~~~
[Responder Core]

; Servers to start
SQL = On
SMB = Off
RDP = On 
Kerberos = On
FTP = On 
POP = On
SMTP = On
IMAP = On
HTTP = Off
HTTPS = On
DNS = On  
LDAP = On
~~~

Then we'll start Responder:
~~~
sudo responder -I eth0 -w
~~~

We can start the relay now; I've thrown 1 IP into a text file to target:
~~~
sudo python3 ntlmrelayx.py -tf ~/Desktop/targets -smb2support
~~~

The attack is all ready, just need to simulate some traffic we can poison:
![Trigger](/assets/posts/socks-relay/trigger.png){: .mx-auto.d-block :}

We see Responder pick up on the requests:
![Responder](/assets/posts/socks-relay/resp.png){: .mx-auto.d-block :}

And ntlmrelayx dumping the local account hashes with the relay:
![Relay Dump](/assets/posts/socks-relay/relay1.png){: .mx-auto.d-block :}

Unfortunately for us, we can guess that the built-in local Administrator account is disabled since its NTLM hash, starting with `31d6c`, looks like the hash for a null/blank password. Usually this indicates the account is disabled (quite common in client environments) and that a custom local administrator account is used. 

This is likely the `radovid` account we saw dumped. This hash is less helpful for us since the account can't obtain admin context over pass-the-hash:
![PTH no admin](/assets/posts/socks-relay/pth-no-admin.png){: .mx-auto.d-block :}

We can successfully authenticate, but we're missing the yellow `(Pwn3d!)` CrackMap uses to denote admin rights. This is because non-RID 500 local accounts have their admin rights "filtered" away when logging in using methods like SMB. The registry key that controls this is `LocalAccountTokenFilterPolicy` found at `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System`. The value is set to `0` on our target, meaning token filtering will take place: 
![LocalAccountTokenFilterPolicy](/assets/posts/socks-relay/filterpolicy.png){: .mx-auto.d-block :}

I do occasionally see machines with this set to `1` meaning PTH still yeilds admin access, but since it's not, we're a little stuck without cracking the hash and claiming the cleartext password. This is a prime situation to use the SOCKS feature of `ntlmrelayx`.

# SOCKS Relay
We'll restart the relay with the `-socks` flag:
~~~
sudo python3 ntlmrelayx.py -tf ~/Desktop/targets -socks -smb2support
~~~

Trigger the LLMNR/NBT-NS requests again, and we see the connection in the relay window, but no local account hashes:
![Relay SOCKS](/assets/posts/socks-relay/relay2.png){: .mx-auto.d-block :}

If we type `socks` into our relay prompt, we can see the active connections available, and if the session has admin rights on the target:
![Relay Sessions](/assets/posts/socks-relay/socks-conns.png){: .mx-auto.d-block :}

Since we have an available session, we can start using `proxychains` to take advantage of our admin rights on the target. Before we do, make sure `/etc/proxychains.conf` is configured to use port `1080`.

### CrackMapExec
CME is usually my tool of choice so lets cover it first. Two things to note when running CME (and other tools) through proxychains with a SOCKS relay:

1. We need to specify domain and username *exactly* how `ntlmrelayx` shows. If you look back at the screenshot of our available sessions, the username for the session shows `REDANIA/VIZIMIR`. We need to be aware of this - for example, if we don't specify `REDANIA` as the domain, CME will default to `REDANIA.local` and the SOCKS session will not be utilizied properly.
2. The password we provide is arbitrary since the SOCKS proxy is going to use the session we gained from relaying auth previously. In my examples, the password used in commands is not the real password.

Knowing this, lets test CME:
![CME Auth](/assets/posts/socks-relay/proxyauth-cme.png){: .mx-auto.d-block :}

We have limited functionality with CME; `smbexec` may work, but `wmiexec` and `atexec` will fail since more than port 445 is used. But, we could still obtain local account hashes with `--sam` and registry secrets with `--lsa` (we could also use `secretsdump.py` directly). We already established that the local account hashes won't help us much here, so lets check for LSA secrets:
![LSA](/assets/posts/socks-relay/proxyauth-lsa.png){: .mx-auto.d-block :}

### smbclient
In some environments I've gotten `smbclient` to work with the SOCKS proxy auth, however, here in my lab it's decided not to:
![smbclient](/assets/posts/socks-relay/proxyauth-smbclient.png){: .mx-auto.d-block :}

If it does work, we could PUT/GET files from the C drive. Useful for potentially uploading procdump or download sensitive files.

### smbmap
Well, if `smbclient` didn't work, `smbmap` won't either... right?
![smbmap](/assets/posts/socks-relay/proxyauth-smbmap.png){: .mx-auto.d-block :}

Wrong, for reasons unknown to me, `smbmap` can read the C drive in my setup, but `smbclient` can't. Maybe I was using smbclient wrong, but like I said, I've had it work before. At any rate, we could `smbmap` to accomplish the same functionality we desired with `smbclient`.

### smbexec
Similar to the oddities with `smbclient`, I've had issues with `smbexec.py` not working in every environment. But we *should* be able to obtain and interactive session using it. We can specify `-no-pass` here since the password we supply when prompted won't matter:
![smbexec](/assets/posts/socks-relay/proxyauth-smbexec.png){: .mx-auto.d-block :}

Seems to be working well here. We should also be able to use `smbexec` through CME with `--exec-method smbexec` if you don't desire an interactive shell.

### What if smbexec fails?
Say `smbexec` did fail on us, is all hope of getting remote command execution lost? Nope, remember when we dumped LSA secrets with CME? We've got the machine account NTLM hash so we can make ourselves a silver ticket and get access that way - actually breaking out of the port 445 restriction currently imposed on us by the SOCKS proxy (no need for proxychains if we go this route).

Here's the important line from the LSA dump:
~~~
REDANIA\NOVIGRAD$:aad3b435b51404eeaad3b435b51404ee:d864dc1f5426f154b74479b5e371a79d:::
~~~

And creation of the silver ticket (assuming you've gotten the domain SID through some other means):
![Silver Ticket](/assets/posts/socks-relay/silvertick.png){: .mx-auto.d-block :}

We set the `KRB5CCNAME` variable, so let's test access using Kerberos auth, outside of our proxychains setup:
![Kerberos Auth](/assets/posts/socks-relay/silver-auth.png){: .mx-auto.d-block :}

:)

Of course, there are other ways you could utilize the SOCKS feature of `ntlmrelayx`, but the flexibility demonstrated here is why I've made this my preferred relay method. 