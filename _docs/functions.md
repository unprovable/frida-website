---
layout: docs
title: Functions
permalink: /docs/functions/
---

We show how to use Frida to inspect functions as they are called, modify their
arguments, and do custom calls to functions inside a target process.

## Setting up the experiment

Create a file `hello.c`:

{% highlight c %}
#include <stdio.h>
#include <unistd.h>

void
f (int n)
{
  printf ("Number: %d\n", n);
}

int
main ()
{
  int i = 0;

  printf ("f() is at %p\n", f);

  while (1)
  {
    f (i++);
    sleep (1);
  }
}
{% endhighlight %}

Compile with:

{% highlight bash %}
$ gcc -Wall hello.c -o hello
{% endhighlight %}

Start the program and make note of the address of `f()` (`0x400544` in the
following example):

{% highlight bash %}
f() is at 0x400544
Number: 0
Number: 1
Number: 2
…
{% endhighlight %}

## Hooking Functions

The following script shows how to hook calls to functions inside a target
process and report back a function argument to you. Create a file `hook.py`
containing:

{% highlight py %}
import frida
import sys

session = frida.attach("hello")
script = session.create_script("""
Interceptor.attach(ptr("%s"), {
    onEnter: function(args) {
        send(args[0].toInt32());
    }
});
""" % int(sys.argv[1], 16))
def on_message(message, data):
    print(message)
script.on('message', on_message)
script.load()
sys.stdin.read()
{% endhighlight %}

Run this script with the address you picked out from above (`0x400544` on our
example):

{% highlight bash %}
$ python hook.py 0x400544
{% endhighlight %}

This should give you a new message every second on the form:

{% highlight py %}
{u'type': u'send', u'payload': 531}
{u'type': u'send', u'payload': 532}
…
{% endhighlight %}

## Modifying Function Arguments

Next up: we want to modify the argument passed to a function inside a target
process. Create the file `modify.py` with the following contents:

{% highlight py %}
import frida
import sys

session = frida.attach("hello")
script = session.create_script("""
Interceptor.attach(ptr("%s"), {
    onEnter: function(args) {
        args[0] = ptr("1337");
    }
});
""" % int(sys.argv[1], 16))
script.load()
sys.stdin.read()
{% endhighlight %}

Run this against the `hello` process (which should be still running):

{% highlight bash %}
$ python modify.py 0x400544
{% endhighlight %}

At this point, the terminal running the `hello process` should stop counting
and always report `1337`, until you hit `Ctrl-D` to detach from it.

{% highlight bash %}
Number: 1281
Number: 1282
Number: 1337
Number: 1337
Number: 1337
Number: 1337
Number: 1296
Number: 1297
Number: 1298
…
{% endhighlight %}

## Calling Functions

We can use Frida to call functions inside a target process. Create the file
`call.py` with the contents:

{% highlight py %}
import frida
import sys

session = frida.attach("hello")
script = session.create_script("""
var f = new NativeFunction(ptr("%s"), 'void', ['int']);
f(1911);
f(1911);
f(1911);
""" % int(sys.argv[1], 16))
script.load()
{% endhighlight %}

Run the script:

{% highlight bash %}
$ python call.py 0x400544
{% endhighlight %}

and keep a watchful eye on the terminal (still) running `hello`:

{% highlight bash %}
Number: 1879
Number: 1911
Number: 1911
Number: 1911
Number: 1880
…
{% endhighlight %}

## Experiment No. 2 - Injecting Strings and Calling a Function

Injecting integers is really useful, but we can also inject strings,
and indeed, any other kind of object you would require for fuzzing/testing.

Create a new file `hi.c`:

{% highlight c %}
#include <stdio.h>
#include <unistd.h>

int
f (const char * s)
{
  printf ("String: %s\n", s);
  return 0;
}

int
main ()
{
  const char * s = "Testing!";

  printf ("f() is at %p\n", f);
  printf ("s is at %p\n", s);

  while (1)
  {
    f (s);
    sleep (1);
  }
}
{% endhighlight %}

In a similar way to before, we can create a script `stringhook.py`, using Frida
to inject a string into memory, and then call the function f() in the following
way:

{% highlight py %}
import frida
import sys

session = frida.attach("hi")
script = session.create_script("""
var st = Memory.allocUtf8String("TESTMEPLZ!");
var f = new NativeFunction(ptr("%s"), 'int', ['pointer']);
    // In NativeFunction param 2 is the return value type,
    // and param 3 is an array of input types
f(st);
""" % int(sys.argv[1], 16))
def on_message(message, data):
    print(message)
script.on('message', on_message)
script.load()
{% endhighlight %}

Keeping a beady eye on the output of `hi`, you should see something along these
lines:

{% highlight bash %}
...
String: Testing!
String: Testing!
String: TESTMEPLZ!
String: Testing!
String: Testing!
...
{% endhighlight %}

Use similar methods, like `Memory.alloc()` and `Memory.protect()` to manipulate
the process memory with ease. Couple this with the python `ctypes` library, and
other memory objects, like `structs` can be created, loaded as byte arrays, and
then passed into functions as pointer arguments.

## Injecting Malicious Memory Objects - Example: sockaddr_in struct

Anyone who has done network programming knows that one of the most commonly
used data structres is the `struct` in C. Here is a naive example of a program
that creates a network socket, and connects to a server over port 5000, and 
announces itself by sending the string `"Hello there!!!"` over the connection.

{% highlight c %}
#include <sys/socket.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <netdb.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <arpa/inet.h> 

int main(int argc, char *argv[])
{
    int sockfd = 0, n = 0, i;
    char recvBuff[1024];
    struct sockaddr_in serv_addr; 
    char *b = (char *)&serv_addr;
    const char *message = "Hello there!!";
    
    if(argc != 2)
    {
        printf("\n Usage: %s <ip of server> \n",argv[0]);
        return 1;
    } 

    printf("connect() is at: %p\n", connect);
    
    memset(recvBuff, '0',sizeof(recvBuff));
    if((sockfd = socket(AF_INET, SOCK_STREAM, 0)) < 0)
    {
        printf("\n Error : Could not create socket \n");
        return 1;
    } 

    memset(&serv_addr, '0', sizeof(serv_addr)); 

    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(5000); 

    if(inet_pton(AF_INET, argv[1], &serv_addr.sin_addr)<=0)
    {
        printf("\n inet_pton error occured\n");
        return 1;
    } 
    printf("\nHere's the serv_addr buffer:\n");
    for (i=0; i< sizeof(serv_addr); i++)
        printf("%02x ", (unsigned char)b[i]);
    
    printf("\nPress ENTER key to Continue\n");  
    getchar(); 
    
    if( connect(sockfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) < 0)
    {
       printf("\n Error : Connect Failed \n");
       return 1;
    } 

    if( send(sockfd , message , strlen(message) , 0) < 0)
    {
        puts("Send failed");
        return 1;
    }

    while ( (n = read(sockfd, recvBuff, sizeof(recvBuff)-1)) > 0)
    {
        recvBuff[n] = 0;
        if(fputs(recvBuff, stdout) == EOF)
        {
            printf("\n Error : Fputs error\n");
        }
    } 

    if(n < 0)
    {
        printf("\n Read error \n");
    } 
    return 0;
}
{% endhighlinght %}

This is fairly standard code, and calls out to any IP address given as the 
first argument. If you run `nc -l 5000` and in another terminal window run
`./client 127.0.0.1`, you should see the message appear in netcat, and also 
able to send messages back to `client` in return.

Now, we can start having some fun - as we saw above, we can inject strings and
pointers into the process. We can do the same by manipulating the struct 
`sockaddr_in` which the program spits out as part of its operation:

{% highlight bash %}
$>./client 127.0.0.1
connect() is at: 0x400780

Here's the serv_addr buffer:
02 00 13 88 7f 00 00 01 30 30 30 30 30 30 30 30 
Press ENTER key to Continue
{% endhighlight %}

If you are not fully familiar with the structure of a struct, there are many
resources online that will tell you what's what. The important bit here are the
bytes `0x1388`, or 5000 in hex. This is our port number (the 4 bytes that
follow are the IP address in hex). If we change this to `0x1389` then we can
re-direct our service to a different point. If we change the next 4 bytes we
can change the IP address that the service points at completely! 

Here's a script to inject the malicious struct into memory, and then hijack the
`connect()` function in `libc.so` to take our new struct as its argument.

Create the file `struct_mod.py` as follows:

{% highlight py %}
#!/usr/bin/python

import frida
import sys

session = frida.attach("client")
script = session.create_script("""
// First, let's give ourselves a bit of memory to put our struct in:
send("Allocating memory and writing bytes...")
var st = Memory.alloc(16);
// Now we need to fill it - this is a bit blunt, but works...
Memory.writeByteArray(ptr(st),[0x02, 0x00, 0x13, 0x89, 0x7F, 0x00, 0x00, 0x01, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30, 0x30]);
// The Interceptor.attach() can find stuff whithout knowing the source module,
// but it's slower, especially over large binaries! YMMV...
    Interceptor.attach(Module.findExportByName(null, "connect"), {
    onEnter: function(args) {
        send("Injecting malicious byte array:");
        args[1] = st;
    }
    //onLeave: function(retval) {
    //   retval.replace(0)        // use this to manipulate the return value
    //}
});

""")
def print_result(message):
            print "[i] Message from script: %s" %(message)
# Here's some message handling..
# [ It's a little bit more meaningful to read as output :-D 
# Errors get [!] and messages get [i] prefixes. ]
def on_message(message, data):
        if (message['type'] == "error"):
		print("[!] "+message['stack'])
	elif (message['type'] == "send"):
		print("[i] "+message['payload'])    
	else:
		print(message)
script.on('message', on_message)
script.load()
sys.stdin.read()
{% endhilight %}

Note that this script demonstrates how the `Module.findExportByName` can be
used to find any function by name in our target. If we can supply a module then
it will be faster on larger binaries, but that is less critical here.

Now, run `./client 127.0.0.1`, in another terminal run `nc -l 5001`, and in a
third terminal run `./struct_mod.py`. Once our script is running, press ENTER
in the `client` terminal window, and netcat should now show the string sent
by the client.

We have successfully hijacked the raw networking by injecting our own data 
object into memory and hooking our process with frida, and using `Intercept`
to do our dirty work in manipulating the function.

This shows the real power of frida - no patching, complicated reversing, nor
difficult hours spent staring at dissassembly without end. 

Here's a quick video demonstrating the above:

    https://www.youtube.com/watch?v=cTcM7R872Ls
