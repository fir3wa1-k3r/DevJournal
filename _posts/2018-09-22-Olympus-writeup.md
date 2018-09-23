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
  <img width="756" height="398" src="https://fir3wa1-k3r.github.io/imgs/olympus_6.png">
</p>
