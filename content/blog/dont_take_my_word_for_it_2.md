---
title: "Don't Take My Word For It (Part 2): Tricking the compiler"
date: 2021-12-26
summary: "Using compiler error messages to accurately determine a type."
---

So far in [Part 1]({{< ref "dont_take_my_word_for_it_1" >}}) we've covered:
* using `typeid` to get the type of a variable, subject to a friendly compiler vendor and losing the top-level const if there is one, and
* using `typeid` to get the dynamic type of something.

The first is useful in a pinch, but not portable, and the second sort of solves a different problem. In the "pinch" situation, we probably just want to know what `T` is and then get one with something else. The compiler generally knows what `T` is - perhaps we can trick it into telling us? Make it do something unreasonable with our mystery type `T` and hopefully the true identity of `T` will appear somewhere in the resulting error message(s).

```c++
#include <iostream>
#include <typeinfo>

int main ()
{
  const auto cpp_string = std::string ("foo");

  // Try to get the type of cpp_string
  int i = cpp_string;
}
```

Compiler output on gcc ((compiler explorer)[https://godbolt.org/z/MPYjTWva7]):
```
<source>: In function 'int main()':
<source>:9:11: error: cannot convert 'const std::__cxx11::basic_string<char>' to 'int' in initialization
    9 |   int i = cpp_string;
      |           ^~~~~~~~~~
      |           |
      |           const std::__cxx11::basic_string<char>```
```

Similarly with `const auto c_string = "foo"` we get `error: invalid conversion from 'const char*' to 'int'`. Note the slightly different error messages: evidently the compiler thinks quite differently about built-in conversions (in our case pointer-to-char and int) than it thinks about user-defined conversions (yes `std::string` from a `char*` counts as user-defined, even though it's in the standard library). In particular, *we have again lost a `const`* in this second message, as we will see shortly.

There is a more reliable trick. We know `decltype()` can get the type of a variable or of an expression, but we need to get that type into an error message. Before, we took a value and did something unreasonable with it, now we want to take a type and do something unreasonable with it. Want to do something unreasonable with types? Use a template :).

First bit:
```c++
template <typename T>
struct Incomplete;
```
`Incomplete` is a template struct that's incomplete - i.e. it's been declared (the compiler knows it exists) but not defined. There's no `{}` next to it, an object of `Incomplete<int>` could never exist because the compiler wouldn't even know how much space one needs.

In the hope that the compiler will give us a particularly readable error message, we now need to put this into action:
```c++
template <typename T>
struct Incomplete;

int main ()
{
  const auto c_string = "foo";

  using T = decltype (c_string);
  Incomplete<T> incomplete;
}
```

And by C++ standards it's not too bad ([compiler explorer](https://godbolt.org/z/5Kf5ohd55)):
```
<source>: In function 'int main()':
<source>:9:17: error: aggregate 'Incomplete<const char* const> incomplete' has incomplete type and cannot be defined
    9 |   Incomplete<T> incomplete;
      |                 ^~~~~~~~~~
```

The key bit is `Incomplete<const char* const> incomplete`. The bit in the angle brackets is our mystery `T`, the type of `c_string`. A second `const` has appeared, because this is a const-pointer-to-a-const-char. In other words, you can't change the pointer (the memory address) and you can't change what it points to (the letter `f` as it happens).

For maximum unfriendliness and minimal typing, we can reduce this down slightly, using more letters and fewer words and typedefs:

```c++
template <typename T> struct X;

int main ()
{
  const auto c_string = "foo";
  X<decltype(c_string)> _;
}
```
Because why wouldn't you call your useless struct `X` and your impossible variable `_`? :)

In this part of the series, we've looked at the using error messages to deduce types. In [Part 3]({{< ref "dont_take_my_word_for_it_3" >}}) we will try to answer a general question about type deduction by generating a fancy table.