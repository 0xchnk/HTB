We start with basic enumeration:

```bash 
nmap -sV -sC 10.129.32.188

echo "10.129.32.188 orion.htb" | sudo tee -a /etc/hosts
```

![](images/Pasted%20image%2020260624211251.png)

We take a look of what we got:

![](images/Pasted%20image%2020260624211951.png)

the only interesting thing right now s the **contact** section, so we move to directory busting:

```bash
gobuster dir -u  http://orion.htb -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt 
```
![](images/Pasted%20image%2020260625004357.png)


We find a login page with:  **Craft CMS 5.6.16**

![](images/Pasted%20image%2020260624215054.png)

We check the cvedetails.com website or on google for any known CVE's for version 5.6.16
link: https://www.cvedetails.com
 and soon enough:
 
 **Craft CMS version 5.6.16** is vulnerable to **CVE-2025-32432**, a critical **unauthenticated Remote Code Execution (RCE)** vulnerability with a **CVSS v3.1 score of 10.0**.
 
 We found the exploit on Metasploit and we run it on our target:
 ```bash
 msfconsole -q
 ```
 ![](images/Pasted%20image%2020260630231105.png)
 
 After a few cd and ls we land on the database credentials:
 
 ![](images/Pasted%20image%2020260630231253.png)
 
 Now we use them to get into the DB and extract or look for more credentials:
 
 ```bash
 mysql -u root -p orion
 #then entered the password
 ```
 ![](images/Pasted%20image%2020260630232157.png)
 
 We crack the hash
 ![](images/Pasted%20image%2020260630232626.png)
 
 And since Adam is a user on the machine we can access the machine through ssh using the founded credentials! 
 ![](images/Pasted%20image%2020260630232903.png)
 once we retireve the system flag;
![](images/Pasted%20image%2020260630232959.png)

 We start looking for a way to escalate our privileges :)
 
 We notice a suspicious open listening port that belongs to telnet
 ![](images/Pasted%20image%2020260630235405.png)
 When we check its version it turns out that it is the 2.7 one that has a [CVE-2026-24061](https://nvd.nist.gov/vuln/detail/CVE-2026-24061)
 with a base CVSS score of 9.8 (critical)
 
 ```bash
 USER="-f root" telnet -a 127.0.0.1
 ```
 And now all is left to do is to retrieve the root flag:
 ![](images/Pasted%20image%2020260630235853.png)
 
Easy machine.

0xchnk