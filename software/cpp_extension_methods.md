# Extension Methods for C++

Use operator overload shenanigans to make a function carrier (Monad...?) invokable on any object using that operator, e.g.

- Illegal: `int a = 2; return(a.IsEven());`
- ...legal? `int a = 2; return a ^ snns::Is(Even);`

Probably not operator ^

You cannot template on function pointers (They have no value yet in a constantly evaluated context!) but you can template on *invokable types* and those can then, in turn, call the function you want (though, again, not templated so maybe it'll be ugly and, frankly, worthless? The point of extension methods is to make things elegant - though: Elegant *at the call site*, it's allowed to be ugly to write them! Better if it's elegant, of course, so do take a look at that, but start by finding out if it is even possible.)

This compiles and the asserts don't fire:

https://godbolt.org/z/eTWfK6bqo

```c++
#include <cassert>
#include <cstdint>
#include <string>

void
func() {};

template<typename Function, typename ... Args>
bool True(Function f, Args ... args)
{
    return f(args...);
}

bool
is_even(double d)
{
    auto integral = (long long)d;
    auto floating = (double)integral;

    if (floating != d)
        return false;

    integral = integral / 2;
    integral = integral * 2;

    return integral == (long long)d;
}

bool are_same(double lhs, std::string str)
{
    return lhs == std::stod(str);
}

/*
        a ^ Do<Divide>(2.0);
*/

int
main()
{
    assert(is_even(4.0));
    assert(!is_even(4.1));
    assert(!is_even(5.0));
    assert(is_even(0.0));

    assert(True(is_even, 2.0));

    assert(True(are_same, 1.0,"1.0"));
}
```