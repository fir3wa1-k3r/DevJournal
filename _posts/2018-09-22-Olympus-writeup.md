---
published: true
---

# HackTheBox Olympus Writeup

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
![2](https://fir3wa1-k3r.github.io/imgs/olympus_2.png)

And concurrently when we run gobuster for directory bruteforcing that resulted in the `index.php` page which is the redirection to the original homepage.<br>
After performing a stego analysis on the images i found nothing. Then i jumped to analyse the response from the web server.Then by looking into the headers, i found that XDebug is enabled in the server, which can allow remote debugging facility for developers.<br>
For more info about XDebug check [here](https://xdebug.org/docs/)<br>
![3](https://fir3wa1-k3r.github.io/imgs/olympus_3.png)



