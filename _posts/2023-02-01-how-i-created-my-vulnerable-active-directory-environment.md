---
title: how i created my vulnerable AD environment
categories:
  - projects
tags:
  - active directory
  - vulnad
pin: false
image:
  path: 'https://n1kkogg.github.io/assets/img/image.png'
  alt: vulnad
published: true
---

# prologue

If you are into cybersecurity you absolutely want to get you personalized active directory lab it's so fun.


It can take much to learn active directory attacks deeply but its so fun trust me its worth it

here's how i built my personal lab :)

# building the lab

the first thing we want to do is creating a new virtual machine with a instance of *windows server* of the version you want 

first we go to the official microsoft website

![img](https://n1kkogg.github.io/assets/img/win10server.PNG)

and download the iso. Then we can just import it to vmware or virtualbox.

Here's a quick tutorial for vmware:

first of all we need to create a new virtual machine and configure it properly to run the newly downloaded iso

![img](https://n1kkogg.github.io/assets/img/vmwarepronewvm.PNG)

then a popup will be created and you will have to select the configuration you want. For now select typical.

then you need to select the iso you just downloaded 

![img](https://n1kkogg.github.io/assets/img/vmwareselectiso.PNG)

now we can click next and select the guest operating system, in your case it's Microsoft Windows - Windows Server 2019 ( or other versions depending on what you chose)

you can also select the location of your virtual machine, i suggest you to not select your primary drive as it is a bit space expensive.

select your maximum disk size in gb and select split virtual disk into multiple files.

you can now customize the hardware of your new vm

i recomend you modify the memory to 4gb if possible and if you want you can also increase or decrease the number of processors and cores.

![img](https://n1kkogg.github.io/assets/img/vmwareoptions.PNG)

now you are ready to start you new vm.

after starting you will have to slect the language of installation, and the keyboard layout

![img](https://n1kkogg.github.io/assets/img/langselect.PNG)

now click install now.

after a bit a new pop up will appear asking you what version of windows you want, the command line or the desktop experience.

Now that's up to you, if you want a desktop experience with a gui then select Window Server 2019 standard evaluation (Desktop Experience) else just install Windows Server 2019 Standard Evaluation.

![img](https://n1kkogg.github.io/assets/img/windowsver.PNG)

I've installed the non gui one, as i dont' really need it.

click next and accept the license terms.


Now it will ask what type of installation you want, since you have vmware you need to click on custom and click next.

now it will actually begin to install :))

after installing you need to setup your administrator password and your domain. How to do that you say? Well it's actually simpler than the gui.

after setting up the admin password write the console command `sconfig`

![img](https://n1kkogg.github.io/assets/img/sconfig.PNG)

as you can see it's menu based 

> (p.s the background could be black or blue doesn't matter).
{: .prompt-tip }

# Setting up the domain

now let's get to the fun part: **setting up the domain**:

first of all let's set up our new computer name

select `2` in sconfig and then enter your computer name, i'll enter MAKIDC.

now select `13` and reboot.

to install the actual active directory you need to first of all install the requred windows features

```powershell
Install-windowsfeature AD-domain-services
Import-Module ADDSDeployment
```

unfortunately you will have to write these by hand if you didn't enable the shared clipboard on vmware, you say "why it's just two commands it isn't really that painful":

```powershell
Install-ADDSForest -CreateDnsDelegation:$false -DatabasePath "C:\\Windows\\NTDS" -DomainMode "7" -DomainName "maki.local" -DomainNetbiosName "maki" -ForestMode "7" -InstallDns:$true -LogPath "C:\\Windows\\NTDS" -NoRebootOnCompletion:$false -SysvolPath "C:\\Windows\\SYSVOL" -Force:$true
```

as you can see it is a bit painful so here's a quick tutorial on how to enable shared clipboard on vmware

first go to vmware and click on Edit virtual machine settings. 

now click on options and then check these two boxes

![img](https://n1kkogg.github.io/assets/img/sharedclip.PNG)

that's it! you have shared clipboard! 

now you should be able to right click to paste this command

```powershell
Install-ADDSForest -CreateDnsDelegation:$false -DatabasePath "C:\\Windows\\NTDS" -DomainMode "7" -DomainName "maki.local" -DomainNetbiosName "maki" -ForestMode "7" -InstallDns:$true -LogPath "C:\\Windows\\NTDS" -NoRebootOnCompletion:$false -SysvolPath "C:\\Windows\\SYSVOL" -Force:$true
```

as you can see i set the domain name to maki.local and the domain net bios name to maki, you should change it to whatever you want.

now reboot the system after it's done and that's it!

# installing vulnerable AD

now our server isn't really hackable as there aren't any common vulnerabilities, to automatically create some we can install this project here called vulnerable AD by safebuffer on github

https://github.com/safebuffer/vulnerable-AD

to install it simply on our system we can just

```powershell
IEX((new-object net.webclient).downloadstring("https://raw.githubusercontent.com/wazehell/vulnerable-AD/master/vulnad.ps1"));
Invoke-VulnAD -UsersLimit 100 -DomainName "maki.local"
```

this simple command downloads the ps1 code and directly loads it in memory with Invoke-Expression (IEX).

> alternatively you can just download it with invoke-webrequest and then import-module like this:
`iwr https://raw.githubusercontent.com/wazehell/vulnerable-AD/master/vulnad.ps1 -o .\vulnad.ps1` then 
`. .\vulnad.ps1` or `Import-Module .\vulnad.ps1` to import it.
{: .prompt-tip }

note the second command i ran that actually invokes the vulnad module, remember to change the **domain name** and the **users limit**, that simply lets the module know how much users need to be generated.

# epilogue

that's it! you now have a vulnerable environment that you can hack! I'd recomend to download bloodhound and run a scan with a user default password `Changeme123!`

here are all the supported attacks for vulnad:

- Abusing ACLs/ACEs
- Kerberoasting
- AS-REP Roasting
- Abuse DnsAdmins
- Password in Object Description
- User Objects With Default password (Changeme123!)
- Password Spraying
- DCSync
- Silver Ticket
- Golden Ticket
- Pass-the-Hash
- Pass-the-Ticket
- SMB Signing Disabled

have fun and thx for reading! <3
