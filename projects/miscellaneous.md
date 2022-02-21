---
layout: page
title: Other Projects
subtitle: Miscellaneous tools I hope someone finds useful
---

This page contains details about smaller tools with random functionaltiy I've created, likely to ease manual or tedious processes.

# aesKrbKeyGen <a href="https://github.com/Tw1sm/aesKrbKeyGen"><img width=40 src="https://github.githubassets.com/images/modules/logos_page/Octocat.png"/></a>
aesKrbKeyGen is a Python3 script to calculate Kerberos keys for Active Directory accounts using a cleartext password. This is a port of Kevin Robertson's PowerShell script, [Get-KerberosAESKey.ps1](https://gist.github.com/Kevin-Robertson/9e0f8bfdbf4c1e694e6ff4197f0a4372).

# Namedaddy <a href="https://github.com/Tw1sm/namedaddy"><img width=40 src="https://github.githubassets.com/images/modules/logos_page/Octocat.png"/></a>

Namedaddy is a tool to quickly manage DNS records via domain registrar API clients. It currently supports the Namecheap, Route53 and GoDaddy APIs, although it's built in an extensible manner. This tool is great for quickly updating A records or initally configuring DNS the first time you use a domain (MX, DKIM, DMARC, SPF), even offering customizable presets for infrastructure you use often.

DNS record commands:
```
Command     Syntax <Required> (Optional)            About
=======     ============================            =====
add         <type> (priority> <name> <value>        Add a DNS record. Priority is only for MX
back                                                Return to main menu
delete      GoDaddy: <record name> (record type)    Deletes GoDaddy DNS records by name or name/type combos. All records matching criteria will be deleted
delete      Namecheap <record id>                   Deletes Namecheap DNS records by record ID
exit                                                Close ConfigDaddy
help                                                Displays help menu
records                                             Display DNS records
stockconfig <config name>                           Configure a new domain with preset records
updateip    <ip addr>                               Updates the IP the address domain points to
```

# HTTPS-MalleableC2-Config <a href="https://github.com/Tw1sm/HTTPS-MalleableC2-Config"><img width=40 src="https://github.githubassets.com/images/modules/logos_page/Octocat.png"/></a>
For many pentests, our Cobalt Strike Malleable C2 profile details are less important than simply just having a profile and SSL setup. Setting C2 profiles up manually is time consuming, while automated tools to do it for you often fail (in my experience) or limit you to one preconfigured profile. This script lets you quickly pick a preconfigured profile (from [rsmudge's git repo](https://github.com/rsmudge/Malleable-C2-Profiles)) for use with your C2 domain and provides you with a Java Keystore and C2 profile ready to initalize your teamserver with.

# LinkedInRecon <a href="https://github.com/Tw1sm/LinkedInRecon"><img width=40 src="https://github.githubassets.com/images/modules/logos_page/Octocat.png"/></a>

LinkedInRecon is basically a quick and dirty Python port of the [GatherContacts](https://github.com/clr2of8/GatherContacts) Burpsuite extension, which collects potential employee names/titles from a Google dork (site:linkedin.com "Acme Corporation"). I occasionally experienced inconsistencies getting the Burp extension running, so built this as an alternative.


# Nessporter <a href="https://github.com/Tw1sm/Nessporter"><img width=40 src="https://github.githubassets.com/images/modules/logos_page/Octocat.png"/></a>

For engagements where you have a group of Nessus scans under a folder, Nessporter allows you to download all scans from the folder in whichever output format. For some reason, Nesssus doesn't have native functionality to download more than one scan at a time.

# Nessmerger <a href="https://github.com/Tw1sm/nessmerger"><img width=40 src="https://github.githubassets.com/images/modules/logos_page/Octocat.png"/></a>

Similar to Nessporter, if you have more than one scan for a single engagement, Nessmerger can be used to combine output `.nessus` files into one `.xlsx` report. This just aggregates the vulnerability data into several spreadsheet tabs with a few summary statistics.
