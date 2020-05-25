Hack The Box — Poison Writeup
==========================================

[Image of Poison](https://github.com/s-farhat/HTB-WriteUps/blob/master/images/Poison.png)

**Personal review**: This Machine took me about three (3) hours to gain root privilege. It was pretty easy however, as I'm trying to avoid working w/o Metasploit and, as the version of Linux deployed on this box contains many vulnerabilities, it was pretty challenging.


### 1. Reconnaissance
First of all, i run a quick nmap scan to figure out which ports are opened and also the service running behind those ports.<br>
```
nmap -sC -sV -O 10.10.10.84 > Quick-scan-Poison.nmap
```
The output of the last command showed the open ports listed below:
* 22/tcp : OpenSSH 7.2 (FreeBSD 20161230; protocol 2.0)
* 80/tcp : Apache httpd 2.4.29 ((FreeBSD) PHP/5.6.32)

Before working on those results, i run a full nmap scan in order to make sure that i didn't miss any other opened ports. <br>
```
nmap -sC -sV -O -p- 10.10.10.84 > Full-scan-Poison.nmap
```
This full scan doesn't report a new open ports:


### 2. Enumeration:
After getting the result of the quick scan, i immidiately start the enumeration phase.
First of all, i opened a browser and i requested the URL **http://10.10.10.84/**.<br>
The response shows that the website serves to test local .php scripts and there's a text field to specify the name of the script and a submit button. the website site also shows 5 files that we can test via the form which are:
* ini.php
* info.php
* listfiles.php
* phpinfo.php 

the 'phpinfo.php' got my interest so i decided to test this script to know more about the webserver's configuration.
After submitting the form, i noticed that the URL is
```
http://10.10.10.84/browse.php?file=phpinfo.php
```
I was pretty sure that this could be vulnerable to LFI exploit and by replacing 'phpinfo.php' by '/etc/passwd' and sending my GET request the browser showed me the content of this file. My hypothesis of LFI exploit is now comfirmed 100%.<br>
The next step is to found how to get into the target host. I kept in my mind that there's a SSH service running on 22/tcp so, the first idea that i thought about was to look into the 'Charix' user (that the passwd file revealed its identity) home folder to look after an 'id_rsa' file which i can use later to login via SSH.<br>
Unfortunately, by requesting 'http://10.10.10.84/browse.php?file=/home/charix/.ssh/id_rsa' the browser returned a permission denied error and then i knew that i have to abandon my hypothesis about 'id_rsa' file.<br>
At this point, i got 2 more options left. The first is to find the password of 'charix' user somewhere on the server and the second is to find a way to write into a file which is stored on the target server and then inject a payload into this file and finally execute it using the browser.<br>
while i was thinking what option should i adopt, i noticied that there's a 'listfiles.php'. I gave it a try by executing by requesting 'http://10.10.10.84/browse.php?file=listfiles.php' using the browser and here's the result below:<br>

```
Array ( [0] => . [1] => .. [2] => browse.php [3] => index.php [4] => info.php [5] => ini.php [6] => listfiles.php [7] => phpinfo.php [8] => pwdbackup.txt )
```
At this point, i hurried to see what can i find into the 'pwdbackup.txt' file and here's below what i found:

```
This password is secure, it's encoded atleast 13 times.. what could go wrong really.. Vm0wd2QyUXlVWGxWV0d4WFlURndVRlpzWkZOalJsWjBUVlpPV0ZKc2JETlhhMk0xVmpKS1IySkVU bGhoTVVwVVZtcEdZV015U2tWVQpiR2hvVFZWd1ZWWnRjRWRUTWxKSVZtdGtXQXBpUm5CUFdWZDBS bVZHV25SalJYUlVUVlUxU1ZadGRGZFZaM0JwVmxad1dWWnRNVFJqCk1EQjRXa1prWVZKR1NsVlVW M040VGtaa2NtRkdaR2hWV0VKVVdXeGFTMVZHWkZoTlZGSlRDazFFUWpSV01qVlRZVEZLYzJOSVRs WmkKV0doNlZHeGFZVk5IVWtsVWJXaFdWMFZLVlZkWGVHRlRNbEY0VjI1U2ExSXdXbUZEYkZwelYy eG9XR0V4Y0hKWFZscExVakZPZEZKcwpaR2dLWVRCWk1GWkhkR0ZaVms1R1RsWmtZVkl5YUZkV01G WkxWbFprV0dWSFJsUk5WbkJZVmpKMGExWnRSWHBWYmtKRVlYcEdlVmxyClVsTldNREZ4Vm10NFYw MXVUak5hVm1SSFVqRldjd3BqUjJ0TFZXMDFRMkl4WkhOYVJGSlhUV3hLUjFSc1dtdFpWa2w1WVVa T1YwMUcKV2t4V2JGcHJWMGRXU0dSSGJFNWlSWEEyVmpKMFlXRXhXblJTV0hCV1ltczFSVmxzVm5k WFJsbDVDbVJIT1ZkTlJFWjRWbTEwTkZkRwpXbk5qUlhoV1lXdGFVRmw2UmxkamQzQlhZa2RPVEZk WGRHOVJiVlp6VjI1U2FsSlhVbGRVVmxwelRrWlplVTVWT1ZwV2EydzFXVlZhCmExWXdNVWNLVjJ0 NFYySkdjR2hhUlZWNFZsWkdkR1JGTldoTmJtTjNWbXBLTUdJeFVYaGlSbVJWWVRKb1YxbHJWVEZT Vm14elZteHcKVG1KR2NEQkRiVlpJVDFaa2FWWllRa3BYVmxadlpERlpkd3BOV0VaVFlrZG9hRlZz WkZOWFJsWnhVbXM1YW1RelFtaFZiVEZQVkVaawpXR1ZHV210TmJFWTBWakowVjFVeVNraFZiRnBW VmpOU00xcFhlRmRYUjFaSFdrWldhVkpZUW1GV2EyUXdDazVHU2tkalJGbExWRlZTCmMxSkdjRFpO Ukd4RVdub3dPVU5uUFQwSwo= 
```
while decoding this text and it's result until getting something that is seems like a password, i thought that maybe it could be a rabbit hole as the box is rated **Medium** severity but i keep doing it because if it's really a rabbit hole, i will just lose 2-3 minutes so i kept decoding until i get the password.<br>
Then, i tried to open an ssh connection with 'Charix' user privilege and i provided the decrypted password and in fact, this has really worked and i got into server with the 'Charix' privilege.<br>

At that moment, i was able to read the user flag and the next step is to run after the root privilege.

### 4. Privilege Escalation:

After getting the user privilege, i noticied that there's a zip file under the home directory. <br>

I managed to transfer the zip file to my own environment and i tried to unzip it however it's protected by a password. As the only password that i found is the password of 'Charix' user i tried it to unlock the zip file and it worked.<br>
The output of the result is non readable so i tried to know more about it by executing the command below.<br>
```
file secret
```

and here's the result below.<br>

```
Non-ISO extended-ASCII text, with no line terminators

```
So i realised that this file isn't an elf and i supposed that it could be a password. Then, i tried to figure out if there's other services running in local by executing: <br>

```
sockstat -4l
```
The result shows that there's a vnc service running locally on port 5901. <br>

At this time, i did a ssh tunneling to access to this service from my proper envoronment by executing. <br>

```
ssh -L 5901:localhost:5901 charix@10.10.10.84
```

and finally, i got access to the target host by executing. <br>


```
vncviewer localhost:5901 -passwd secret
```
which allowed me to get root privilege. <br>


**Box Rooted !!!**

