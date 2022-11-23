# THM SecDSM-Dec22-MiniCTF Writeup

​		22<sup>nd</sup> December 2022

 



### Description:

This machine...

### Difficulty:

`easy`

# Hosts File
```
sudo nano /etc/hosts
```
add a line with the remote IP and the dns entry

```
<IP> secdsm.thm
```

# Enumeration

Start with basic and advanced NMAP scans on the box.
```
nmap -v secdsm.thm
```
![image](https://user-images.githubusercontent.com/43767555/203445905-4d04de66-d01c-48f0-9407-229137a38694.png)
![image](https://user-images.githubusercontent.com/43767555/203445915-0b21ff59-a63c-4ecb-8c61-27616b5fb5ab.png)

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



# Privilege Escalation

