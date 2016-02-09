### Solved by superkojiman and barrebas

barrebas did the intial enumeration and found CesarFTP 0.99 running on the target. CesarFTP is a windows FTP server, so we were up against a Windows server.

```
Starting Nmap 6.00 ( http://nmap.org ) at 2014-10-18 14:04 CEST
Nmap scan report for 10.66.66.3
Host is up (0.015s latency).
Not shown: 998 filtered ports
PORT     STATE SERVICE
21/tcp   open  ftp
3389/tcp open  ms-wbt-server

Nmap done: 1 IP address (1 host up) scanned in 6.41 seconds
bas@tritonal:~/documents$ ftp 10.66.66.3
Connected to 10.66.66.3.
220 CesarFTP 0.99g Server Welcome !
Name (10.66.66.3:bas):
```

There are actually exploits readily available for this, but I had also written one in the past and was able to dig it up. This target also had RDP open so I connected to it and found that it was running Windows XP Pro. This was important in order to properly customize the exploit to jump to the payload.

However, the FTP server was always down. My guess is that people were crashing it while fuzzing or trying an exploit that didn't work. I'm not entirely sure why they didn't come up with a mechamis to relaunch CesarFTP, or perhaps they were unable to keep up with all the fuzzing. To get around it, they provided the virtual machine of the target for download.

At that point, there was no reason to exploit CesarFTP. I simply booted the virtual machine up with a CrunchBang Linux live CD, navigated to C:\Documents and Settings\FTP\Desktop and found the flag in a file called ThisIsIt.txt

![](/images/2014/defcamp/misc400/01.png)

The flag is **Caesar Salad**

