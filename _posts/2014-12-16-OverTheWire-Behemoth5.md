---
layout: post
title: OverTheWire - Behemoth 5
---

Site: http://overthewire.org/wargames/behemoth/ <br>
Level: 5<br>
Situation: Reading Assembly Exercise?<br><br>

So let us start how we usually do. We are going to create our temp directory, check file arch type, run our application.

{% highlight bash %}
behemoth5@melinda:~$ mkdir /tmp/bh5
behemoth5@melinda:~$ cd /tmp/bh5
{% endhighlight %}

{% highlight bash %}
behemoth5@melinda:/tmp/bh5$ file /behemoth/behemoth5
/behemoth/behemoth5: setuid ELF 32-bit LSB  executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=cc9166a1934734dac9f39544c1383cef7bfa84d8, not stripped
{% endhighlight %}

{% highlight bash %}
behemoth5@melinda:/tmp/bh5$ /behemoth/behemoth5
behemoth5@melinda:/tmp/bh5$ 
{% endhighlight %}

Not much is being told to us here. We will now do some information grabbing. We will grab the global offset table and ltrace the application.

{% highlight bash %}
behemoth5@melinda:/tmp/bh5$ objdump -R /behemoth/behemoth5

/behemoth/behemoth5:     file format elf32-i386

DYNAMIC RELOCATION RECORDS
OFFSET   TYPE              VALUE 
08049ffc R_386_GLOB_DAT    __gmon_start__
0804a00c R_386_JUMP_SLOT   fgets
0804a010 R_386_JUMP_SLOT   fclose
0804a014 R_386_JUMP_SLOT   rewind
0804a018 R_386_JUMP_SLOT   htons
0804a01c R_386_JUMP_SLOT   fseek
0804a020 R_386_JUMP_SLOT   perror
0804a024 R_386_JUMP_SLOT   malloc
0804a028 R_386_JUMP_SLOT   __gmon_start__
0804a02c R_386_JUMP_SLOT   exit
0804a030 R_386_JUMP_SLOT   strlen
0804a034 R_386_JUMP_SLOT   __libc_start_main
0804a038 R_386_JUMP_SLOT   ftell
0804a03c R_386_JUMP_SLOT   fopen
0804a040 R_386_JUMP_SLOT   memset
0804a044 R_386_JUMP_SLOT   sendto
0804a048 R_386_JUMP_SLOT   atoi
0804a04c R_386_JUMP_SLOT   socket
0804a050 R_386_JUMP_SLOT   gethostbyname
0804a054 R_386_JUMP_SLOT   close
{% endhighlight %}

We gain a bit of information here we can clearly see that socket and gethostbyname is being called. As well as fgets and seek. So we can assume a few things.

1. A file is being read and parsed
2. A socket of some kind is being opened.

{% highlight bash %}
behemoth5@melinda:/tmp/bh5$ ltrace /behemoth/behemoth5
__libc_start_main(0x804873d, 1, 0xffffd784, 0x8048960 <unfinished ...>
fopen("/etc/behemoth_pass/behemoth6", "r")         = 0
perror("fopen"fopen: Permission denied
)                                    = <void>
exit(1 <no return ...>
+++ exited (status 1) +++
{% endhighlight %}

From this we now know what file is being opened. To our surprise it is the password file for the next level.

Note. It is failing to open the file due to ltrace running the application with behemoth5 permissions. Under normal execution it can access the file.

Because we know that a socket is being and we know the odds of it being a server socket are slim do to the fact that the applications closes. We have a general idea of what to look for. Now we simply load up our debugger and disassemble to see what’s going on.

{% highlight bash %}
behemoth5@melinda:/tmp/bh5$ gdb /behemoth/behemoth5
(gdb) unset env LINES
(gdb) unset env COLUMNS
(gdb) set disassembly-flavor intel
(gdb) disassemble main
Dump of assembler code for function main:
-- SNIP --
   0x08048829 <+236>:	call   0x8048620 <gethostbyname@plt>
   0x0804882e <+241>:	mov    DWORD PTR [esp+0x30],eax
   0x08048832 <+245>:	cmp    DWORD PTR [esp+0x30],0x0
   0x08048837 <+250>:	jne    0x8048851 <main+276>
   0x08048839 <+252>:	mov    DWORD PTR [esp],0x8048a1f
   0x08048840 <+259>:	call   0x8048560 <perror@plt>
   0x08048845 <+264>:	mov    DWORD PTR [esp],0x1
   0x0804884c <+271>:	call   0x8048590 <exit@plt>
   0x08048851 <+276>:	mov    DWORD PTR [esp+0x8],0x0
   0x08048859 <+284>:	mov    DWORD PTR [esp+0x4],0x2
   0x08048861 <+292>:	mov    DWORD PTR [esp],0x2
   0x08048868 <+299>:	call   0x8048610 <socket@plt>
   0x0804886d <+304>:	mov    DWORD PTR [esp+0x34],eax
   0x08048871 <+308>:	cmp    DWORD PTR [esp+0x34],0xffffffff
   0x08048876 <+313>:	jne    0x8048890 <main+339>
   0x08048878 <+315>:	mov    DWORD PTR [esp],0x8048a2d
   0x0804887f <+322>:	call   0x8048560 <perror@plt>
   0x08048884 <+327>:	mov    DWORD PTR [esp],0x1
-- SNIP --

-- SNIP --
   0x08048911 <+468>:	mov    eax,DWORD PTR [esp+0x34]
   0x08048915 <+472>:	mov    DWORD PTR [esp],eax
   0x08048918 <+475>:	call   0x80485f0 <sendto@plt>
   0x0804891d <+480>:	mov    DWORD PTR [esp+0x38],eax
   0x08048921 <+484>:	cmp    DWORD PTR [esp+0x38],0xffffffff
   0x08048926 <+489>:	jne    0x8048940 <main+515>
   0x08048928 <+491>:	mov    DWORD PTR [esp],0x8048a39
   0x0804892f <+498>:	call   0x8048560 <perror@plt>
   0x08048934 <+503>:	mov    DWORD PTR [esp],0x1
   0x0804893b <+510>:	call   0x8048590 <exit@plt>
   0x08048940 <+515>:	mov    eax,DWORD PTR [esp+0x34]
   0x08048944 <+519>:	mov    DWORD PTR [esp],eax
   0x08048947 <+522>:	call   0x8048630 <close@plt>
   0x0804894c <+527>:	mov    DWORD PTR [esp],0x0
   0x08048953 <+534>:	call   0x8048590 <exit@plt>
-- SNIP --
End of assembler dump.

{% endhighlight %}

We can confirm that indeed a socket is being made. Also we see sendto function sendto is used to broadcast a message on a stateless connection a.k.a "UDP". We are only missing the host and the port now. If we had a good debugger we would be able to see the data in each memory address in our disassembly. We could iterate over all the addresses and look at memory. However to make things quick we run strings.

{% highlight bash %}
behemoth5@melinda:/tmp/bh5$ strings /behemoth/behemoth5
-- SNIP --
QVh=
D$L1
[^_]
/etc/behemoth_pass/behemoth6
fopen
localhost
gethostbyname
socket
1337
sendto
-- SNIP --
{% endhighlight %}

Looks like localhost and 1337. So to listen on the port we are going to use netcat.

{% highlight bash %}
behemoth5@melinda:/tmp/bh5$ nc -ul 1337
{% endhighlight %}

{% highlight bash %}
behemoth5@melinda:/tmp/bh5$ /behemoth/behemoth5
behemoth5@melinda:/tmp/bh5$ 
{% endhighlight %}

{% highlight bash %}
behemoth5@melinda:/tmp/bh5$ nc -ul 1337
[OMITTED]
{% endhighlight %}

There is our password.