---
comments: true
layout: post
title:  std::bind for unknown variable
date:   2015-07-09 01:28
---
I find the std::bind function a very cool thing, it allows storing functions with different parameters in the same list. But you can even do bind calls within bind calls so that at the moment you call the function it will first execute another function and then passing the result as parameter to the function that you call.

But what if you need to bind a parameter to the function but you don't know yet what variable should be bound? You could use a pointer as parameter and then later set the pointer to the variable of course. But although I will use a pointer I will have it dereference automatically before the function is actually called.
<!--more-->

First of all it is important to note that the type of the variable has to be known. I will use void* so that you can reuse the code for any type, but when binding you need to know of which type the variable will be.

Let's start with defining the function that we will call. This could be any function that uses your chosen type, but I like templates so why not use a templated function that can accept any type? Also let's make the parameter a reference to point out that we can get even change the variable that the pointer will be pointing to.
{% highlight c++ %}
template <typename T>
void print(T& obj) {
    std::cout << obj << std::endl;
}
{% endhighlight %}

I said I was going to use a pointer, but the function takes the object itself so it has to be dereferenced somewhere along the way. Although the std library provides functions like std::plus for the '+' operator, there is no function for the dereference operator. So we must define our own function. I will again use a template and a void* so that I can reuse this function for any type, but you can make it simpler if it is specifically for one type.
{% highlight c++ %}
template <typename T>
T& dereference(void* obj) {
    return *static_cast<T*>(obj);
}
{% endhighlight %}

We finally arrived at the binding part. I will demonstrate that the same pointer can be used for different types by binding two different print functions. As you can see the code has no idea what variable is going to be passed as parameter of the print function at the moment of binding, the variable hasn't even been declared yet. Note that you must use std::ref to pass a reference to the pointer, otherwise it gets copied and we can't make it point to a variable afterwards.
{% highlight c++ %}
void* ptr = nullptr;
auto func1 = std::bind(print<int>, dereference<int>, std::ref(ptr));
auto func2 = std::bind(print<std::string>, dereference<std::string>, std::ref(ptr));
{% endhighlight %}

To finish it off we just have to set our variables and call the function.
{% highlight c++ %}
int i = 5;
ptr = static_cast<void*>(&i);
func1();

std::string s = "Hello";
ptr = static_cast<void*>(&s);
func2();
{% endhighlight %}

That's it! As you can see, you did not need to have access to the variables at the moment of binding. I hope this ends up being useful to somebody someday.
