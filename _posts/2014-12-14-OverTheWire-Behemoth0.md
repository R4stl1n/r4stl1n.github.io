---
layout: post
title: OverTheWire - Behemoth 0
---

Site: http://overthewire.org/wargames/behemoth/ <br>
Level: 0 <br>
Situation: Simple memory read. <br><br>

{% highlight bash %}
behemoth0@melinda:~$ file /behemoth/behemoth0

/behemoth/behemoth0: setuid ELF 32-bit LSB  executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=4c2e0281c9220ac21b55994f2a2408fe3c6693ac, not stripped
{% endhighlight %}
We know it is 32-bit application and Intel Arch.
Let’s run it and get the output.
{% highlight bash %}
behemoth0@melinda:~$ /behemoth/behemoth0
Password: aaaa
Access denied..
{% endhighlight %}

Alright should be simple enough lets go ahead and open this up in GDB.

{% highlight bash %}
behemoth0@melinda:~$ gdb -q /behemoth/behemoth0
Reading symbols from /behemoth/behemoth0...(no debugging symbols found)...done.
(gdb) set disassembly-flavor intel
(gdb) disassemble main
Dump of assembler code for function main:
   -- SNIP --
   0x08048602 <+96>:	call   0x8048470 <__isoc99_scanf@plt>
   0x08048607 <+101>:	lea    eax,[esp+0x1f]
   0x0804860b <+105>:	mov    DWORD PTR [esp],eax
   0x0804860e <+108>:	call   0x8048440 <strlen@plt>
   0x08048613 <+113>:	mov    DWORD PTR [esp+0x4],eax
   0x08048617 <+117>:	lea    eax,[esp+0x1f]
   0x0804861b <+121>:	mov    DWORD PTR [esp],eax
   0x0804861e <+124>:	call   0x804857d <memfrob>
   0x08048623 <+129>:	lea    eax,[esp+0x1f]
   0x08048627 <+133>:	mov    DWORD PTR [esp+0x4],eax
   0x0804862b <+137>:	lea    eax,[esp+0x2b]
   0x0804862f <+141>:	mov    DWORD PTR [esp],eax
   0x08048632 <+144>:	call   0x80483f0 <strcmp@plt>
   0x08048637 <+149>:	test   eax,eax
   0x08048639 <+151>:	jne    0x8048665 <main+195>
   0x0804863b <+153>:	mov    DWORD PTR [esp],0x8048771
   0x08048642 <+160>:	call   0x8048420 <puts@plt>
   0x08048647 <+165>:	mov    DWORD PTR [esp+0x8],0x0
   0x0804864f <+173>:	mov    DWORD PTR [esp+0x4],0x8048782
   -- SNIP --
{% endhighlight %}

Looking at this we find there is a good ole strcmp. It’s a c function used to compare two strings. We know that our data will be on the stack when this compare occurs so we can easily debug this and grab the password. Works for this case but generally won't work in most.  

{% highlight bash %}
(gdb) break *0x08048632
Breakpoint 1 at 0x8048632
(gdb) run
Starting program: /games/behemoth/behemoth0 
Password: aaa

Breakpoint 1, 0x08048632 in main ()
(gdb) x/s $eax
0xffffd67b:	"aaa"
(gdb) x/s $esp
0xffffd650:	"{\326\377\377o\326\377\377p\326\377\377\322\202\004\b \207\004\b8\207\004\bM\207\004\b\346m\353[OMITTED]"
{% endhighlight %}

There is our password, we could also do this on the test function which we can see is taking the EAX register.