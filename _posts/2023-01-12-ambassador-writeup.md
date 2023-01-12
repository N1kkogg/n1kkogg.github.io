---
title: Ambassador Writeup
categories: [HTB, linux]
tags: [nmap, lfi, consul]
image:
  path: https://cybersaradiyel.com/content/images/2022/12/Ambassador-min-crop-1.png
  alt: hackthebox ambassador
---

# enumeration

It is recommended to run a `rustscan` scan as it is a fast and reliable tool for performing `network reconnaissance`. It's a good practice to use this tool in order to gather information about the target's *open ports* and *services*.

I always like to run a scan with the `-g` option to concatenate open ports with a comma like this:

```bash
┌─[makider@makider-virtualbox]─[~/htb/broscience]
└──╼ $rustscan -a 10.129.218.98 -g
10.129.218.98 -> [22,80,3000,3306]
```

I can then utilize this information by incorporating the identified ports into nmap command for further enumeration.

Here is a template of the command i use for the enumeration process:

```bash
sudo nmap -sVCS -oN nmap/initial $IP -p $PORTS
```

This command utilizes the `-sVCS option`, which allows for a more comprehensive scan of the target's open ports. Let's break this down:

- `-sV` option to enable version detection
- `-sC` option to enable *default* scripts (same as --script default)
- `-sS` option to enable SYN Stealth Scan that only forges a SYN packet to purposefully *not* complete the TCP handshake (logging evasion)


the `-oN` option is useful to log the output of nmap to a file (in this case to the `initial` file in the `nmap` directory)


Here is the output of this command:

```bash
Nmap scan report for 10.129.218.98
Host is up (0.049s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 29:dd:8e:d7:17:1e:8e:30:90:87:3c:c6:51:00:7c:75 (RSA)
|   256 80:a4:c5:2e:9a:b1:ec:da:27:64:39:a4:08:97:3b:ef (ECDSA)
|_  256 f5:90:ba:7d:ed:55:cb:70:07:f2:bb:c8:91:93:1b:f6 (ED25519)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Ambassador Development Server
|_http-generator: Hugo 0.94.2
|_http-server-header: Apache/2.4.41 (Ubuntu)
3000/tcp open  ppp?
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.0 302 Found
|     Cache-Control: no-cache
|     Content-Type: text/html; charset=utf-8
|     Expires: -1
|     Location: /login
|     Pragma: no-cache
|     Set-Cookie: redirect_to=%2Fnice%2520ports%252C%2FTri%256Eity.txt%252ebak; Path=/; HttpOnly; SameSite=Lax
|     X-Content-Type-Options: nosniff
|     X-Frame-Options: deny
|     X-Xss-Protection: 1; mode=block
|     Date: Wed, 05 Oct 2022 16:42:41 GMT
|     Content-Length: 29
|     href="/login">Found</a>.
|   GenericLines, Help, Kerberos, RTSPRequest, SSLSessionReq, TLSSessionReq, TerminalServerCookie: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 302 Found
|     Cache-Control: no-cache
|     Content-Type: text/html; charset=utf-8
|     Expires: -1
|     Location: /login
|     Pragma: no-cache
|     Set-Cookie: redirect_to=%2F; Path=/; HttpOnly; SameSite=Lax
|     X-Content-Type-Options: nosniff
|     X-Frame-Options: deny
|     X-Xss-Protection: 1; mode=block
|     Date: Wed, 05 Oct 2022 16:42:10 GMT
|     Content-Length: 29
|     href="/login">Found</a>.
|   HTTPOptions: 
|     HTTP/1.0 302 Found
|     Cache-Control: no-cache
|     Expires: -1
|     Location: /login
|     Pragma: no-cache
|     Set-Cookie: redirect_to=%2F; Path=/; HttpOnly; SameSite=Lax
|     X-Content-Type-Options: nosniff
|     X-Frame-Options: deny
|     X-Xss-Protection: 1; mode=block
|     Date: Wed, 05 Oct 2022 16:42:15 GMT
|_    Content-Length: 0
3306/tcp open  mysql   MySQL 8.0.30-0ubuntu0.20.04.2
|_sslv2: ERROR: Script execution failed (use -d to debug)
| mysql-info: 
|   Protocol: 10
|   Version: 8.0.30-0ubuntu0.20.04.2
|   Thread ID: 13
|   Capabilities flags: 65535
|   Some Capabilities: DontAllowDatabaseTableColumn, InteractiveClient, SupportsCompression, ODBCClient, SupportsLoadDataLocal, Support41Auth, IgnoreSigpipes, SupportsTransactions, Speaks41ProtocolOld, FoundRows, LongColumnFlag, LongPassword, SwitchToSSLAfterHandshake, IgnoreSpaceBeforeParenthesis, ConnectWithDatabase, Speaks41ProtocolNew, SupportsAuthPlugins, SupportsMultipleStatments, SupportsMultipleResults
|   Status: Autocommit
|   Salt: 6!C&~\x17S%\#\x13\x02()uB_ j\x0D
|_  Auth Plugin Name: caching_sha2_password
|_tls-alpn: ERROR: Script execution failed (use -d to debug)
|_ssl-cert: ERROR: Script execution failed (use -d to debug)
|_ssl-date: ERROR: Script execution failed (use -d to debug)

```

Okay, port `22,80,3000` and `3306` are open

- port `22` is a simple `SSH` server
- port 80 is a `Apache` server it should contain a website, and thanks to the `-sC` option for nmap we have managed to retrieve the http-title and generator:

```bash
|_http-title: Ambassador Development Server
|_http-generator: Hugo 0.94.2
```

so we know that this server was made by using Hugo generator version 0.94.2 and is a Developement Server, intresting...

- port 3000 as we will verify later is a `grafana` server

```
Grafana allows you to query, visualize, alert on, and understand your metrics no matter where they are stored. Create, explore, and share beautiful dashboards with your team and foster a data-driven culture.
```

- port 3306 is a mysql server


 First of all i started by enumerating the "low hanging fruit" mysql by checking on internet for known exploits regarding this version (`MySQL 8.0.30-0ubuntu0.20.04.2`) but nothing came up.

 So i tried bruteforcing the login with the default user (`mysql`) and with the metasploit `auxiliary/scanner/mysql/mysql_login` module, but nothing, we need more information, maybe we need a user?

 Next one in the list is port 80 that actually catched my eye's attention due to the intresting name it had.

 let's see...






