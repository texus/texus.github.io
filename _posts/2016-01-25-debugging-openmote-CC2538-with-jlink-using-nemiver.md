---
comments: true
layout: post
title:  Debugging on OpenMote-CC2538 with J-Link (using nemiver)
date:   2016-01-25 19:02
---

After trying for several days to get debugging working in Eclipse, I was told that it would be easier to use nemiver. I tried it and worked on the first try without any problems. This post will show the easy steps necessary to debug the code running on the OpenMote hardware.

My setup is the following: a linux pc (running Arch Linux), the [SEGGER J-Link EDU](https://www.segger.com/j-link-edu.html), the [OpenMote-CC2538](http://www.openmote.com/shop/openmote-cc2538.html) and the [OpenBase](http://www.openmote.com/hardware/openbase.html) to connect the mote to the pc. Also an [ARM-JTAG-20-10](https://www.olimex.com/Products/ARM/JTAG/ARM-JTAG-20-10/) adaptor was used to connect the J-Link to the JTAG on the OpenBase.
<!--more-->

The first thing to do is start the J-Link GDB Server which you can [download from the SEGGER website](https://www.segger.com/jlink-software.html). I am using V4.96f as I faced some problems introduced in V5.02b, but these issues should have already been fixed so the latest version should also work.
{% highlight bash %}
JLinkGDBServer -device CC2538SF53
{% endhighlight %}

The J-Link GDB server will be listening to new GDB connections on port 2331 by default.

In another terminal we can then start the GDB session, but how easy this is depends on whether you are using the [OpenMote firmware](https://github.com/OpenMote/firmware) or not to write your program.

### Using the firmware

When your program is written with the OpenMote firmware you just have to execute the nemiver target in the Makefile. So after running "make" to build the program, running "make nemiver" will do everything for you: start the nemiver debugger, flash your program to the OpenMote and break at the beginning of your program.

### Without the firmware

First create a file (name it e.g. cc2538_gdb.gdb) with the following contents:
{% highlight text %}
target remote localhost:2331
monitor interface JTAG
monitor endian little
monitor speed auto
monitor flash device = CC2538SF53
monitor flash breakpoints = 1
monitor flash download = 1
monitor reset
load
break main
continue
{% endhighlight %}

Secondly we setup the GDB connection and upload your program to the J-Link GDB server. The command to run is the ARM GDB, the debugger for the ARM GCC compiler that you used to compile the binary. On Arch Linux this program is called arm-none-eabi-gdb but it may have a different name in other linux distributions (e.g. in Ubuntu the package is called gdb-arm-none-eabi). The parameter to the "\-\-command" option is the file you just created and the "program.elf" is the binary that you should have build.
{% highlight bash %}
arm-none-eabi-gdb --batch --command=cc2538_gdb.gdb program.elf
{% endhighlight %}

Finally you can start nemiver and start debugging your program.
{% highlight bash %}
nemiver --remote=localhost:2331 --gdb-binary=/usr/bin/arm-none-eabi-gdb program.elf
{% endhighlight %}
