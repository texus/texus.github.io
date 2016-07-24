---
layout: post
title:  Converting string to lowercase at compile time with c++14
date:   2015-07-28 00:05
---
With the addition of the [constexpr](http://en.cppreference.com/w/cpp/language/constexpr) keyword in c++11 it is possible to calculate the result of function calls on compile time. I knew that it was possible to e.g. count the brackets in a string, but I wanted to take things a little bit further, I wanted to change the string.

I failed at doing this about a year ago, but now that I just updated to gcc 5.2, I decided to give it another shot.
<!--more-->

The first question that has to be answered is how to return the resulting string. On compile time we can't use types like std::string or std::vector and the parameters can't be changed. You could use [std::initializer_list](http://en.cppreference.com/w/cpp/utility/initializer_list)&lt;char&gt; as the return type which allows directly constructing an std::string from the lowercase string literal so that you can write code like this (edit: this does not work at compile time, see update).
{% highlight c++ %}
std::string str{to_lower("TEST")};
{% endhighlight %}

But it is not that easy to check the code with static asserts. That is why I use a const [std::array](http://en.cppreference.com/w/cpp/container/array)&lt;char, N&gt; as return type in the code below which allows verifying every character at compile time.

The code consists of three functions: one that converts a single character, one helper function that makes the conversion of the string and finally the function that you have to call. It was easy to see that I needed to use a variadic template pack expansion to construct the return value in one line, but this was the thing that seemed impossible. How do you change an N-character string into N parameters? I found the answer when I came across the [std::integer_sequence](http://en.cppreference.com/w/cpp/utility/integer_sequence) type that was added in c++14. The std::make\_integer\_sequence will create a sequence of which the first argument is the type and the other arguments are a sequence of numbers from 0 to N-1. That was all I needed to get the thing working.
{% highlight c++ %}
#include <utility>
#include <array>

constexpr char to_lower_helper_char(const char c) {
    return (c >= 'A' && c <= 'Z') ? c + ('a' - 'A') : c;
}

template <unsigned N, typename T, T... Nums>
constexpr const std::array<char, N> to_lower_helper(const char (&str)[N], std::integer_sequence<T, Nums...>) {
    return {to_lower_helper_char(str[Nums])...};
}

template <unsigned N>
constexpr const std::array<char, N> to_lower(const char (&str)[N]) {
    return to_lower_helper(str, std::make_integer_sequence<unsigned, N>());
}
{% endhighlight %}

The following code can be used to confirm that all the code runs on compile time:
{% highlight c++ %}
int main()
{
    constexpr auto arr = to_lower("TEST");
    static_assert(arr[0] == 't', "ERROR");
    static_assert(arr[1] == 'e', "ERROR");
    static_assert(arr[2] == 's', "ERROR");
    static_assert(arr[3] == 't', "ERROR");
}
{% endhighlight %}


<b>Update</b> <em><small>(4 Aug 2015)</small></em>

After some experimenting it seems like the method does not work well with std::initializer\_list&lt;char&gt; and the std::array was only used for testing and isn't that useful. So I looked for an alternative way to return the data. I basically wanted to return a raw array, but any c++ programmer should knows that you can't do that, so I had to use a simple wrapper instead and changed the return types of the functions to array\_wrapper&lt;N&gt;.
{% highlight c++ %}
template <std::size_t N>
struct array_wrapper {
    const char arr[N];
};
{% endhighlight %}

The only problem is that no matter what my function returns, it is not as user-friendly as I hoped. The following line will always run entirely at runtime, instead of the to_lower function call being run at compile time.
{% highlight c++ %}
std::string str{to_lower("TEST").arr};
{% endhighlight %}

In order to execute it on compile time the line must be split:
{% highlight c++ %}
constexpr auto temp = to_lower("TEST");
std::string str{temp.arr};
{% endhighlight %}
