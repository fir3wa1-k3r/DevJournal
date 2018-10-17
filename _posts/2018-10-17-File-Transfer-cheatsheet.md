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
<br>
	You can use the Python's SimpleHTTPServer module for this particular technique. First you need to fire up the
    python's HTTP server using the command `python -m SimpleHTTPServer`. The `-m` is a switch to specify the
    module python and there you can run this command directly on your shell. And this server serves the files in
    the current directory where it is running.
    <br>
	<p align="center">
  		<img width="614" height="39" src="https://fir3wa1-k3r.github.io/imgs/file_1.png">	
	</p>
	<br>
    This command is executed on the machine from which you want to transfer files to your machine.
	By default this server will run port 8000 you can even specify the custom port by just appending the
    port at the end. In the below example, the server serves on port 80.
    <br>
    `python -m SimpleHTTPServer 80`
    <br>
    And on the remote machine use the `wget` command to download the file from the server.
    <br>
    <p align="center">
  		<img width="886" height="198" src="https://fir3wa1-k3r.github.io/imgs/file_2.png">
  	</p>
    <br>
    <br>
    
* **Python FTP Server**
<br>
<br>
	For this technique, you need to install the python package `pyftpdlib` if not installed.
    <br>
    
    > To know more about installing packages click [here](https://packaging.python.org/tutorials/installing-packages/). 
    
    Then use the command `python -m pyftpdlib` to start the ftp server. By default the server runs on port 2121
    and this can even be customized using the `-p` switch.
    <br>
	<p align="center">
		<img width="800" height="136" src="https://fir3wa1-k3r.github.io/imgs/file_4.png">
	</p>
    <br>
   	This server provides anonymous login facility, and can also include custom username and password.
	<p align="center">
		<img width="686" height="249" src="https://fir3wa1-k3r.github.io/imgs/file_5.png">
	</p>
<br>
<br>

* **Twistd FTP Server**
<br>
<br>
	You need to install twistd program to use this technique. You can do this by running `sudo apt-get install twistd` command on your terminal or with any equivalent package manager.
    You can start the twistd FTP server by running `twistd ftp` command on the terminal. The service starts and runs in the background. By default the twistd service runs on port 2121, we can customize the port. This also allows anonymous login and hence the credentials can also be changed if needed. The twistd FTP service uses `/usr/local/ftp` as the default root directory for the file server. You can modify it by using the switch `-r` or `--root=` and provide the path.
	<p align="center">
		<img width="566" height="172" src="https://fir3wa1-k3r.github.io/imgs/file_6.png">
	</p>
<br>
<br>

* **Nginx server**
<br>
<br>
	You need to have an nginx server on the machine where you want to accept the files. Then you just need to configure a bit to work appropriately. For the Ubuntu system, the configuration files will be under the `/etc/nginx/` directory. We actually need to create a new file here its called **transfer_files** under `/etc/nginx/sites-available/` and add the below code in it.
<br>
{% highlight ruby %} 
server {
		listen 8421 default_server;
        location / {
        		root /dev/shm;
                dav_methods PUT;
       	}
}
{% endhighlight %}
<br>
You customize the port and the root directory by replacing the port and the directory path in the above code.
Then we need to create a symbolic link of the created file to the `/etc/nginx/site-enabled` directory.

`ln -s /etc/nginx/sites-available/transfer_files /etc/nginx/sites-enabled`