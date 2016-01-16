---
comments: true
layout: post
title:  Using parallel_for from Intel TBB
date:   2015-11-17 15:20
---

This post will show an example of the parallel_for function from the <abbr title="Intel Threading Building Blocks">Intel TBB</abbr> library. Every part of the code is well commented to explain what it does. Below the example you will find other explanations on how to improve the code and how to avoid the bad programming practices that were used in the example to keep it simple.
<!--more-->

### parallel_for example
{% highlight c++ %}
#include <iostream>
#include <vector>

// Intel TBB has split the declaration of its classes and functions in different headers so that you only
// include the part you need. We need tbb::blocked_range and tbb::parallel_for in the code so we only need
// to include their corresponding headers.
#include <tbb/parallel_for.h>
#include <tbb/blocked_range.h>

// We have one global list on which we will do calculations. We immediately initialize the list with 10000 zeros.
std::vector<double> data(10000, 0);

// This function contains the code without parallelization.
// The function is not used, I just included it to show what the code will do.
// The algorithm will just increment all elements in the list by 1.
void algorithm_without_tbb()
{
    for (size_t i = 0; i < data.size(); ++i)
        data[i] += 1;
}

// This is the functor which Intel TBB will use inside the tbb::parallel_for function.
// A functor is just a struct/class with a call operator (operator()). Having this operator allows an object
// of this type to behave like a function. Using the call operator allows Intel TBB to work with both classes and functions.
struct IncrementElements
{
    // This function will be called inside tbb::parallel_for. What the range contains is entirely up to Intel TBB,
    // but you can expect it to be [0,9999] if you only use one thread (since there are 10000 elements).
    // If you use 2 threads then Intel TBB could call this function twice, once with [0,4999] and once with [5000,9999].
    void operator()(const tbb::blocked_range<size_t>& range) const
    {
        // Instead of looping over [0,9999] we loop over the range given by Intel TBB.
        // The first element can be retrieved with range.begin() and range.end() will return one after
        // the last element (so that you have to use < instead of <=).
        // The 'range' that we use here was the parameter of this function.
        for (size_t i = range.begin(); i < range.end(); ++i)
            data[i] += 1;
    }
};

int main()
{
    // We just execute all the code 5 times
    for (size_t t = 0; t < 5; ++t)
    {
        // Calling this function instead of running the code below results in the same result if
        // the parallel code is implemented correctly.
        //algorithm_without_tbb();

        // To use Intel TBB we first have to create a range object. This is not the same one as what Intel TBB passes
        // to the functor, but the ranges passed to the functor will always be subranges of this one. So
        // eventually all elements in this range will be processed by Intel TBB.
        // As we want to run the code on all elements in the list, which we usually want, the range is [0,9999]
        tbb::blocked_range<size_t> range(0, data.size());

        // We have to create a temporary instance of the functor class. The 'func' object will behave like
        // a function and by running func(...) which tbb::parallel_for does, the IncrementElements::operator()(...)
        // will be executed. I use 'auto' as type since you shouldn't bother about the exact type of the object.
        auto func = IncrementElements();

        // At this point, nothing has happened yet. By calling tbb::parallel_for, the code in the functor
        // will be executed (on several threads each with their own subrange) so that after this call all
        // elements in the list will have been incremented.
        tbb::parallel_for(range, func);
    }

    // Check to make sure that all elements in the list have been incremented 5 times
    for (size_t i = 0; i < data.size(); ++i)
    {
        if (data[i] != 5)
        {
            std::cout << "ERROR" << std::endl;
            return 1;
        }
    }

    std::cout << "SUCCESS" << std::endl;
    return 0;
}
{% endhighlight %}

### Removing temporary variables
The 'range' and 'func' variables in the main function don't have to be created explicitly, I only did it like that to explain each part separately. So the following lines from the code
{% highlight c++ %}
tbb::blocked_range<size_t> range(0, data.size());
auto func = IncrementElements();
tbb::parallel_for(range, func);
{% endhighlight %}

can be written in one line:
{% highlight c++ %}
tbb::parallel_for(tbb::blocked_range<size_t>(0, data.size()), IncrementElements());
{% endhighlight %}

### Removing the global variable
Global variables are a very bad programming practice and they should be avoided where possible.

So let's change the code so that we no longer rely on a global vector:
{% highlight c++ %}
#include <iostream>
#include <vector>
#include <tbb/parallel_for.h>
#include <tbb/blocked_range.h>

// The global 'data' variable no longer exists in this example.
// The variable will instead be created inside the main function.

class IncrementElements
{
public:

    // Since there is no more global 'data' variable, the operator() will not be able to access the data (which has
    // been created in the main function). To solve this, the main function will pass a pointer to the data to
    // this class which we will copy in our private 'm_data' variable. We have to use a pointer or reference
    // instead of copying the vector because otherwise the original vector will never be changed.
    // The private 'm_data' member has to be a pointer since Intel TBB will internally copy this class a few times.
    IncrementElements(std::vector<double>* dataPtr)
    {
        m_data = dataPtr;
    }

    // It would of course have been easier if the vector could directly be passed as an argument to the
    // call operator, but since this function is called internally by Intel TBB we have no control over its
    // parameters and the vector should thus already be passed earlier, which is usually done in the constructor.
    void operator()(const tbb::blocked_range<size_t>& range) const
    {
        for (size_t i = range.begin(); i < range.end(); ++i)
            (*m_data)[i] += 1; // Note that we now have to use (*m_data)[i] instead of data[i]
    }

    // The 'm_data' variable could just as well be public but it is not needed outside this class
    // and it is better to keep it local instead of allowing other parts of the code to access it.
    // The variable will just hold a pointer to the 'data' vector that we create in the main function.
private:
    std::vector<double>* m_data;
};

int main()
{
    // The data variable is now created locally inside the main function.
    std::vector<double> data(10000, 0);

    for (size_t t = 0; t < 5; ++t)
    {
        // This time the functor gets a pointer to the data as parameter to its constructor.
        // Other than that the code does exactly the same.
        auto func = IncrementElements(&data);
        tbb::parallel_for(tbb::blocked_range<size_t>(0, data.size()), func);

        // The code above could again be written in a single line without the temporary 'func' variable:
        // tbb::parallel_for(tbb::blocked_range<size_t>(0, data.size()), IncrementElements(&data));
    }
}
{% endhighlight %}

### Using the initializer list
I know that many beginning programmers don't know how to use the [initializer list](http://en.cppreference.com/w/cpp/language/initializer_list) so I just initialized the m_data variable inside the constructor itself. It is however a good habit to always use the initializer list when possible. There are even cases where you have to use it and can't assign the value inside the constructor.

The constructor of the IncrementElements looks like this when we use the initializer list:
{% highlight c++ %}
IncrementElements(std::vector<double>* dataPtr)
    : m_data(dataPtr)
{
}
{% endhighlight %}

### Final code
When removing all comments we end up with this short example which you should now be able to understand:
{% highlight c++ %}
#include <vector>
#include <tbb/parallel_for.h>
#include <tbb/blocked_range.h>

struct IncrementElements
{
    IncrementElements(std::vector<double>* dataPtr) :
        m_data{dataPtr}
    {
    }

    void operator()(const tbb::blocked_range<size_t>& range) const
    {
        for (size_t i = range.begin(); i < range.end(); ++i)
            (*m_data)[i] += 1;
    }

private:
    std::vector<double>* m_data;
};

int main()
{
    std::vector<double> data(10000, 0);
    for (size_t t = 0; t < 5; ++t)
        tbb::parallel_for(tbb::blocked_range<size_t>(0, data.size()), IncrementElements(&data));
}
{% endhighlight %}

### More complex code
Although the example above is a very simple one, all code that just executes something on every element of a list has extremely similar code. The for loop that you have in your non-parallel version can almost literally be copied to the call operator of the functor, you just have to change the start and end of your for loop. After that just make sure that the operator has access to all needed variables and you are done.
