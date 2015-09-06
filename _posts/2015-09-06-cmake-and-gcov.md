---
comments: true
layout: post
title:  CMake and Gcov
date:   2015-09-06 03:04
---
While trying to use Gcov for my project I noticed that it doesn't work well together with CMake. The problem is that CMake generates files named file.cpp.gcno while gcov searches for file.gcno. Luckily, there is an easy solution.
<!--more-->

First things first, in order to use Gcov in CMake you have to pass some flags to g++ so something similar to the following should be in your CMakeLists.txt
{% highlight CMake %}
set(CMAKE_CXX_FLAGS "-g -O0 -Wall -fprofile-arcs -ftest-coverage")
{% endhighlight %}

This would cause the *.cpp.gcno files to be generated next to your object files. Assuming your cpp files were inside a src subdirectory, you will find these files in a folder like build/src/CMakeFiles/project_name.dir.

But we didn't want this *.cpp.gcno extension, so the solution is to also add the following line in your CMakeLists.txt
{% highlight CMake %}
set(CMAKE_CXX_OUTPUT_EXTENSION_REPLACE 1)
{% endhighlight %}

That's it! By adding that one line your files will just have the *.gcno extension.

Now we can run our tests. After running the tests you can run gcov in the folder where you want to create your *.gcov files. The code below is meant to be executed in a new empty subdirectory inside your build folder and assumes that your source files were in a src directory.
{% highlight bash %}
gcov ../../src/*.cpp --object-directory ../src/CMakeFiles/project_name.dir/
{% endhighlight %}
