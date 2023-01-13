---
title: Ambassador Writeup
categories: [HTB, linux]
tags: [nmap, lfi, consul]
pin: true
image:
  path: https://cybersaradiyel.com/content/images/2022/12/Ambassador-min-crop-1.png
  alt: hackthebox ambassador
---

## enumeration

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

 Next one in the list is port `80` that actually catched my eye's attention due to the intresting name it had.

 let's see...

 okay it seems to be a developement server but nothing stands out.

Based on this, i decided to run a `nikto` scan to further enumerate the webpage for potential vulnerabilities.

> `Nikto` is an open-source web server scanner that is used to identify potential vulnerabilities in web servers. It can be used to detect a wide range of vulnerabilities including outdated software versions, misconfigurations, and potentially dangerous files or programs.

```bash
nikto -h 10.129.218.98

- Nikto v2.1.5
---------------------------------------------------------------------------
+ Target IP:          10.129.218.98
+ Target Hostname:    10.129.218.98
+ Target Port:        80
+ Start Time:         2022-10-05 18:50:24 (GMT2)
---------------------------------------------------------------------------
+ Server: Apache/2.4.41 (Ubuntu)
+ Server leaks inodes via ETags, header found with file /, fields: 0xe46 0x5e7a7c4652f79 
+ The anti-clickjacking X-Frame-Options header is not present.
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ IP address found in the 'location' header. The IP is "127.0.1.1".
+ OSVDB-630: IIS may reveal its internal or real IP in the Location header via a request to the /images directory. The value is "http://127.0.1.1/images/".
+ Allowed HTTP Methods: GET, POST, OPTIONS, HEAD 
+ OSVDB-3092: /sitemap.xml: This gives a nice listing of the site content.
+ OSVDB-3268: /images/: Directory indexing found.
+ OSVDB-3268: /images/?pattern=/etc/*&sort=name: Directory indexing found.
+ 6544 items checked: 0 error(s) and 8 item(s) reported on remote host
+ End Time:           2022-10-05 18:55:56 (GMT2) (332 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```


intresting we found the `sitemap.xml` file, for those who don't know what it is, it's basically a xml file that lists out all the files exposed on the webpage.

it's very important since it can prevent you from using other enumeration tools like `feroxbuster` or `gobuster` to bruteforce directories.


**Here is sitemap.xml**

```xml
<urlset>
<url>
<loc>https://example.org/</loc>
<lastmod>2022-03-10T19:01:57+00:00</lastmod>
</url>
<url>
<loc>https://example.org/posts/</loc>
<lastmod>2022-03-10T19:01:57+00:00</lastmod>
</url>
<url>
<loc>
https://example.org/posts/welcome-to-the-ambassador-development-server/
</loc>
<lastmod>2022-03-10T19:01:57+00:00</lastmod>
</url>
<url>
<loc>https://example.org/categories/</loc>
</url>
<url>
<loc>https://example.org/tags/</loc>
</url>
</urlset>
```

With the information gathered from the this file we can now proceed to search for vulnerabilities and potentially interesting files on the target page.

**After some time, i noticed this intresting welcome post:**

```markdown
**Welcome to the Ambassador Development Server**
March 10, 2022

Hi there! This server exists to provide developers at Ambassador with a standalone development environment. When you start as a developer at Ambassador, you will be assigned a development server of your own to use.

**Connecting to this machine**

Use the `developer` account to SSH, DevOps will give you the password.
```

This welcome post provides valuable information about the purpose and setup of the development server. I also attempted to connect to the server using the developer account by using common passwords, but without success.

So i decided to move on to the next port: `3000`

