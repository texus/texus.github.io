---
comments: true
layout: post
title:  Cross-platform semi-transparent windows
date:   2015-06-25 01:04
---
In the [previous post]({% post_url 2015-06-20-cross-platform-custom-shaped-windows %}) I talked about custom shaped windows, in this post I will explain how to create semi-transparent windows on Windows, Linux and Mac OS X. The full code can be found in the same [github repository](https://github.com/texus/TransparentWindows). Each operating system will again be discussed separately as they require different code.
<!--more-->

The alpha variable used below is an unsigned char which means its value is between 0 and 255.

### Windows
On windows you just have to make two function calls. The [SetWindowLong](https://msdn.microsoft.com/en-us/library/windows/desktop/ms633591.aspx) call makes the window layered, which is needed in order to set the transparency. Then you call the [SetLayeredWindowAttributes](https://msdn.microsoft.com/en-us/library/windows/desktop/ms633540.aspx) function to set the alpha value.

To use these functions you just have to include windows.h, you don't have to link to any libraries like on linux or mac.
{% highlight c++ %}
SetWindowLong(hWnd, GWL_EXSTYLE, GetWindowLong(hWnd, GWL_EXSTYLE) | WS_EX_LAYERED);
SetLayeredWindowAttributes(hWnd, 0, alpha, LWA_ALPHA);
{% endhighlight %}

### Linux
On linux the code is a little bit more complicated. We must first get the atom identifier of the opacity property with [XInternAtom](http://tronche.com/gui/x/xlib/window-information/XInternAtom.html). Then we set the property with [XChangeProperty](http://tronche.com/gui/x/xlib/window-information/XChangeProperty.html). Don't forget to include X11/Xatom.h and add -lX11 to your linker settings in order to use these functions.

There is something strange going on with XChangeProperty at first sight. The 32 looks like the amount of bits, but there is a small catch. Since a long is 64 bit on a 64-bit linux system, it might look like we are passing too many bytes. But if you read the documentation very carefully you find that the data should be a char (8-bit) when passing 8, short (16-bit) when passing 16, but long (32-bit or 64-bit) when passing 32. So the number doesn't always exactly equal the amount of bits. 

Also we can't just get away with passing 8 and the address of alpha as parameters, since the atom we want to assign to is not a char, hence the conversion to the unsigned long first.

And then we end with calling XFlush to send the request to the X server and XCloseDisplay to free all the memory associated with the display. 
{% highlight c++ %}
Display* display = XOpenDisplay(NULL);
Atom property = XInternAtom(display, "_NET_WM_WINDOW_OPACITY", false);
if (property != None)
{
    unsigned long opacity = (0xffffffff / 0xff) * alpha;
    XChangeProperty(display, wnd, property, XA_CARDINAL, 32, PropModeReplace, (unsigned char*)&opacity, 1);
    XFlush(display);
}
XCloseDisplay(display);
{% endhighlight %}

### Mac OS X
Mac forces us to use Objective-C or Swift (I used Objective-C here), so we can't have the implementation in the same .cpp file as the Windows and Linux version. The code below can be placed in a .mm file. Cocoa is needed so you should also add "-framework Cocoa" to the linker flags.

The code is extremely simple though. The alpha value has to be a float between 0 and 1 here so we must first convert our char to the proper format. Then we just set it with calling the setAlphaValue on the window (which has type [NSWindow](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/ApplicationKit/Classes/NSWindow_Class/)*). The window also has to be told that it is not opaque to make the semi-transparency work, so we pass NO to setOpaque.
{% highlight obj-c %}
CGFloat opacity = alpha / 255.0f;
[window setAlphaValue:opacity];
[window setOpaque:NO];
{% endhighlight %}
