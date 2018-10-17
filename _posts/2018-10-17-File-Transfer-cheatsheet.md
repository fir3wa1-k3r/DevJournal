---
published: true
---
### There are a lot many ways to tranfer a file from one machine to a remote or a local machine in a Linux terminal. Here are a few of them which may help you when you are stuck. Before starting i would like to thank **ippsec** for all these tricks. Thank you man, Kudos to you!!!
<br>
<br>
For this post i will use a simple text file which is called as **text.file**
<br>
<br>
* **Python Http Server**
<br>
	You can use the Python's SimpleHTTPServer module for this particular technique. First you need to fire up the
    python's HTTP server using the command `$python -m SimpleHTTPServer`. The `-m` is a switch to specify the
    module python and there you can run this command directly on your shell.
    <p align="center">
  		<img width="614" height="39" src="https://fir3wa1-k3r.github.io/imgs/file_1.png">	
	</p>
	<br>
    This command is executed on the machine from which you want to transfer files to your machine.
	By default this server will run port 8000 you can even specify the custom port by just appending the
    port at the end. In the below example, the server serves on port 80.
    <br>
    `$python -m SimpleHTTPServer 80` 
    <br>
    
    