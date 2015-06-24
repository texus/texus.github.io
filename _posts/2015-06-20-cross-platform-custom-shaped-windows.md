---
comments: true
layout: post
title:  Cross-platform custom-shaped windows
date:   2015-06-20 17:45
---
This post will explain how to create transparent (non-rectangular) windows on Windows, Linux and Mac OS X. The shape of the window will be determined by an image with a transparent background. If the pixel is transparent then it will not be part of the window. It is assumed that the size of the image and the size of the window are the same.

Each operating system requires different code and will be discussed separately. Below you will find code snippets, but for a full example check this <a href="https://github.com/texus/TransparentWindows">github project</a> which creates a window with a custom shape with SFML.
<!--more-->

In the code snippets below get_window, get_pixel_color, image_width and image_height are just placeholders. How you get these properties is up to you. In my example I used SFML, but the code below does not depend on it.

<h3>Windows</h3>
On Windows we just have to create a region in the wanted shape and use the <a href="https://msdn.microsoft.com/en-us/library/aa930600.aspx">SetWindowRgn</a> function to set it. To get this region we will combine multiple regions with <a href="https://msdn.microsoft.com/en-us/library/aa922002.aspx">CombineRgn</a>.

Since we want our region to be based on an image, we must adapt the region pixel by pixel. It might be slightly more performant to detect bigger rectangles in the image so that less regions have to be created, but the effort in doing so is probably larger than the speed gain.

All needed functions come from the Windows API, which means that windows.h has to be included.

Here is the code with explanations in comments:
{% highlight c++ %}
// Get the window handle from somewhere
HWND hWnd = get_window();

// Create a region that has the size of the entire window
HRGN hRegion = CreateRectRgn(0, 0, image_width, image_height);

// Loop over every pixel in the image
for (unsigned int y = 0; y < image_height; y++)
{
    for (unsigned int x = 0; x < image_width; x++)
    {
        // Check in some way if the pixel is transparent
        if (get_pixel_color(image, x, y).a == 0)
        {
            // Create a 1x1 region that corresponds to the pixel in the image
            HRGN hRegionPixel = CreateRectRgn(x, y, x+1, y+1);

            // Remove the pixel from the region
            CombineRgn(hRegion, hRegion, hRegionPixel, RGN_XOR);

            // Free the memory of the pixel region
            DeleteObject(hRegionPixel);
        }
    }
}

// Set the region of the window (last parameter tells it to redraw the window)
SetWindowRgn(hWnd, hRegion, true);

// Free the memory of the region
DeleteObject(hRegion);
{% endhighlight %}

<h3>Linux</h3>
On linux, non-rectangular windows are not available directly in the X11 library, you will need to use the <a href="http://www.x.org/releases/X11R7.6/doc/libXext/shapelib.html">X Nonrectangular Window Shape Extension Library</a> which is part of the Xext library. On some systems the code will work, while on others it is simply not supported (the availability of the shape extension will be queried at runtime). When linking this code you will have to add "-lX11 -lXext" to the linker flags.

Except for checking the availability, the code is broadly equivalent to the windows version.
{% highlight c++ %}
// Open a connection to the X server on the default display
Display* display = XOpenDisplay(NULL);

// Check if the shape extension is supported
int event_base;
int error_base;
if (XShapeQueryExtension(display, &event_base, &error_base))
{
    // Get the window handle from somewhere
    Window wnd = get_window();

    // Create a black & white pixmap (depth=1) with the size of the window
    Pixmap pixmap = XCreatePixmap(display, wnd, image_width, image_height, 1);

    // Create a graphics context (without any special values)
    GC gc = XCreateGC(display, pixmap, 0, NULL);

    // Make the entire pixmap white
    XSetForeground(display, gc, 1);
    XFillRectangle(display, pixmap, gc, 0, 0, image_width, image_height);

    // Set the color to black for further draw calls
    XSetForeground(display, gc, 0);

    // Loop over every pixel in the image
    for (unsigned int y = 0; y < image_height; y++)
    {
        for (unsigned int x = 0; x < image_width; x++)
        {
            // Check in some way if the pixel is transparent
            if (get_pixel_color(image, x, y).a == 0)
            {
                // Draw a black 1x1 rectangle on the pixmap at the location of the pixel
                XFillRectangle(display, pixmap, gc, x, y, 1, 1);
            }
        }
    }

    // Set the shape of the window based on the pixmap (window shape = white part of pixmap)
    XShapeCombineMask(display, wnd, ShapeBounding, 0, 0, pixmap, ShapeSet);

    // Free the memory of the graphics context and pixmap
    XFreeGC(display, gc);
    XFreePixmap(display, pixmap);

    // Send the requests to the X server
    XFlush(display);
}

// Free all the memory associated with the display
XCloseDisplay(display);
{% endhighlight %}

<h3>
Mac OS X</h3>
Getting custom shapes on Mac is very different from the way we did it on Windows and Linux. You will not have to set any region, you can just tell the window to only be opaque where you draw on it. So by just drawing the image, the window would get the shape of the image. That may sound great, but it introduces a problem when trying to use OpenGL. When drawing with OpenGL you clear the entire screen before drawing, therefore the window would always have a rectangle shape. To work around that you must clear with a transparent color.

Mac forces us to use Objective-C or Swift (I used Objective-C here), so we can't have the implementation in the same .cpp file as the Windows and Linux version. The code below can be placed in a .mm file. Cocoa is needed so you should also add "-framework Cocoa" to the linker flags.

Other tutorials show that in order to get a window with a custom shape you have to subclass <a href="https://developer.apple.com/library/mac/documentation/Cocoa/Reference/ApplicationKit/Classes/NSWindow_Class/">NSWindow</a>. I will show how to do it without inheriting from it, since you might not have access to the window class directly (e.g. when trying to change the shape of a window created with SFML).

{% highlight obj-c %}
// Get the window handle from somewhere
NSWindow* wnd = get_window();

// Tell the window that some parts have to be transparent
GLint opaque = 0;
[[[wnd contentView] openGLContext] setValues:&opaque forParameter:NSOpenGLCPSurfaceOpacity];
[wnd setBackgroundColor:[NSColor clearColor]];
[wnd setOpaque:NO];
{% endhighlight %}

But the above code is not enough if you want to use OpenGL. The <a href="https://developer.apple.com/library/mac/documentation/Cocoa/Reference/ApplicationKit/Classes/NSOpenGLView_Class/index.html">NSOpenGLView</a> class overrides the isOpaque function of <a href="https://developer.apple.com/library/mac/documentation/Cocoa/Reference/ApplicationKit/Classes/NSView_Class/index.html">NSView</a> and makes it return YES while we need it to return NO. In Objective-C we can change the implementation of the function from outside the class. Add the following code somewhere and the NSOpenGLView class and all its subclasses will have this implementation. 
{% highlight obj-c %}
@implementation NSOpenGLView (Opaque)
-(BOOL)isOpaque {
    return NO;
}
@end
{% endhighlight %}

That's it. Now just clear the window with a fully transparent color and draw the image on it.
