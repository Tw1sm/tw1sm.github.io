---
layout: page
title: Other Projects
subtitle: Miscellaneous tools someone may find useful
---

This page contains details about smaller tools with random functionailtiy I've created, likely to ease manual or tedious processes.

# Namedaddy  
<img width=60 href="https://github.com/Tw1sm/namedaddy" src="https://github.githubassets.com/images/modules/logos_page/Octocat.png"/>

 Namedaddy is a tool to quickly manage DNS records via domain registrar API clients. It currently supports the Namecheap and GoDaddy APIs, although it's built in an extensible manner. This tool is great for quickly updating A records or initally configuring DNS the first time you use a domain (MX, DMIK, DMARC, SPF), even offering customizable presets for records you use often.

 ### Domain commands
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


# LinkedInRecon
<img width=60 href="https://github.com/Tw1sm/LinkedInRecon" src="https://github.githubassets.com/images/modules/logos_page/Octocat.png"/>

LinkedInRecon is basically a quick and dirty Python port of the [GatherContacts](https://github.com/clr2of8/GatherContacts) Burpsuite extensio which collects potential employee names/titles from a Google dork (site:linkedin.com "Acme Corporation"). I occasionally experienced inconsistencies getting the Burp extension running, so built this as an alternative.

# Nessus Realted Scripts

## Nessporter
<img width=60 href="https://github.com/Tw1sm/nessporter" src="https://github.githubassets.com/images/modules/logos_page/Octocat.png"/>

For engagements where you have a group of Nessus scans under a folder, Nessporter allows you to download all scans from the folder in whichever output format. For some reason, Nesssus doesn't have native functionality to download more than one scan at a time.

## Nessmerger
<img width=60 href="https://github.com/Tw1sm/nessmerger" src="https://github.githubassets.com/images/modules/logos_page/Octocat.png"/>

Similar to Nessporter, if you have more than one scan for a single engagement, Nessmerger can be used to combine output `.nessus` files into one `.xlsx` report. This just aggregates the vulnerability data into several spreadsheet tabs with a few summary statistics.