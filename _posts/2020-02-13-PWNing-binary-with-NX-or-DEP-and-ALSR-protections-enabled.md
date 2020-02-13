---
published: true
---

## Pwning binary with NX/DEP protections enabled.

Hola people, today lets try to pwn a binary which has NX/DEP protection and the machine has ALSR enabled. I have written a simple C program which utilizes the vulnerable function gets to read the user input. The user input is read into the variable 'a' which is an array of characters of size 20 bytes.

<br>
	<p align="center">
  		<img width="614" height="39" src="https://fir3wa1-k3r.github.io/imgs/pwn_1.png">	
	</p>
<br>

When we run the file command on the compiled binary we see that its a 64 bit ELF and is not stipped which means that the debugging symbols are enabled. And the checksec command tells that only the Non executable stack protection (NX) is enabled which means that we can't just put shellcode on the stack and execute it. If we try to do it, we will end up in segmentation fault.

<br>
	<p align="center">
  		<img width="614" height="39" src="https://fir3wa1-k3r.github.io/imgs/pwn_2.png">	
	</p>
<br>

Let's run the binary and see what it does. It just ask for the user input and then prints out "Thanks!".

<br>
	<p align="center">
  		<img width="614" height="39" src="https://fir3wa1-k3r.github.io/imgs/pwn_3.png">	
	</p>
<br>
 
 Now let's test whether we have a buffer overflow or not (even though we know it is).
 
<br>
	<p align="center">
  		<img width="614" height="39" src="https://fir3wa1-k3r.github.io/imgs/pwn_4.png">
	</p>
<br>
 
 And yes! we received a 'Segmentation fault' which means that we are trying to read/write to some part of the memory where we don't have permission for it.
 
 So, we can find the offset until which we can fill out junk data and whatever is sent after it will overwrite the return address. Let's use gdb to find the offset.
 
<br>
	<p align="center">
  		<img width="614" height="39" src="https://fir3wa1-k3r.github.io/imgs/pwn_5.png">
	</p>
<br>

We create a cyclic pattern of 100 characters in length and input that to our vulnerable binary.

<br>
	<p align="center">
  		<img width="614" height="39" src="https://fir3wa1-k3r.github.io/imgs/pwn_6.png">
	</p>
<br>

The offset is at 40. So, we can fill up junk until 40 character and after which the return address will be modified by data specified by the attacker. Now we have the control of the instruction pointer.

We also see that ALSR is enabled on the server which is why libc's base address gets changed everytime when we run the ldd command on the vulnerable binary.

<br>
	<p align="center">
  		<img width="614" height="39" src="https://fir3wa1-k3r.github.io/imgs/pwn_7.png">
	</p>
<br>

So, we can try to leak the real address of one of the functions in libc library so that we can calculate the libc's base address and even call other libc function just by knowing their base address. Isn't that cool!!
We use Return Oriented Programming (ROP) method in this post to bypass the NX/DEP protection. It is one of the best attacks which you can use to break NX protection.
From the source code of the vulnerable program we see that puts and gets are the libc function that are called. So, we can try to leak the real address of puts from the Global Offset Table (GOT) and then use that address to calcuate the libc's base address.
For that we need to understand what is PLT and GOT. You can read more about it here "https://www.technovelty.org/linux/plt-and-got-the-key-to-code-sharing-and-dynamic-libraries.html"

For now, i will quickly explain what do they do. Before a function from the dynamically linked library gets called, its real address is unknown. So, when it is first called, there is a slow lookup function called Procedural Linkage Table (PLT) which will calculate the real address of that function and updates it in the GOT table. So, we leverage this activity and try to leak the real address of the function.

Lets, get into action now. 