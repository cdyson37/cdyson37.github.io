---
title: "Don't Take My Word For It (Part 1): Getting the type dynamically"
date: 2021-12-25
summary: "Using `typeid` to determine an object's type at runtime."
---
Ever wanted to know what type something is in C++, but your editor can't or won't tell you (or you don't trust it...?) You've probably been in some templated function, nested a few levels deep, gone to get a coffee, come back and forgotten quite how it fits together and asked yourself "what exactly is T anyway"? Don't take my word or anyone else's - If in doubt, ask the compiler (or the runtime).

In Python we're used to doing simply
```py
>>> type("foo")
<class 'str'>
```

Types are objects. `type(some_object)` returns an object that happens to be the type of `some_object`. Since that type is itself an object, you can print it. Informative and helpful.

No such luck in C++: types are not objects. If you try to do `std::cout << std::string` you'll get an angry compiler (`std::string` is not an object, it's a type, so you can't just pass it to some overload of `operator<<`).

A bit of Googling later, you might come across the `typeid` operator which returns a `std::type_info` object, which has a `name()` function. So far this appears pretty helpful. Giving it a test drive:
```c++
#include <iostream>
#include <typeinfo>

int main ()
{
  const auto c_string = "foo";
  const auto cpp_string = std::string ("foo");

  // What's the type of c_string?
  std::cout << typeid(c_string).name() << '\n';

  // What's the type of cpp_string?
  std::cout << typeid(cpp_string).name() << '\n';
}
```

Possible output ([compiler explorer](https://godbolt.org/z/9sPo4zW75)) if you use gcc:
```
PKc
NSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE
```

There's something there, but you wouldn't call it easy to read! And we're already firmly into implementation-defined behaviour. You'll get something else that might be more-or-less readable. At least the second bit at least has the word "string" in it.

Fortunately for gcc users you can demangle this (yes that's a technical term) with a command-line tool:
```
$ c++filt -t PKc
char const*
$ c++filt -t NSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE
std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >
```

With gcc it's also possible to do this on the fly in C++ (see a nice SO answer [here](https://stackoverflow.com/questions/281818/unmangling-the-result-of-stdtype-infoname)), but this is strictly vendor-specific.

So we've got something that works, on one compiler at least, and with a little bit of effort prints out the type of a variable. Well, not quite... `typeid` actually gives you the type of an expression. It doesn't have the special fudge `decltype` has, wherein `decltype(x)` means "the type of `x`" and anything that isn't just a variable name, such as `decltype((x))`, means "the type of the expression".

In our case, both variables should have a top-level const (i.e. `char const * const` and `const std::string`) but `typeid` loses the top-level qualifier for us.

I'm a stranger to the Windows programming world, but I downloaded Visual Studio Community 2022 and ran the same program - here's the output:
```
char const * __ptr64
class std::basic_string<char,struct std::char_traits<char>,class std::allocator<char> >
```
Actually quite helpful! It still loses the top-level const on the string (that's inevitable because that's how `typeid` works) but at least it doesn't need decoding.

There's another use for `typeid` that we haven't covered here but deserves a mention: it can be used for RTTI purposes. If you've got a base class `Base` and one or more derived classes (and you've remembered your virtual destructor in the base class), `typeid` can tell you which derived class a reference-to-base refers to:

```c++
#include <iostream>
#include <typeinfo>

struct Base
{
  virtual ~Base() {};
};

struct D1 : Base {};
struct D2 : Base {};

int main ()
{
  D2 derived;
  Base &ptr_to_base = derived;

  std::cout << std::boolalpha;
  std::cout << (typeid (ptr_to_base) == typeid (D1)) << '\n';
  std::cout << (typeid (ptr_to_base) == typeid (D2)) << '\n';
}
```

Output ([compiler explorer](https://godbolt.org/z/bPrc9Moec)):
```
false
true
```

So far we've looked at the uses of `typeid`. In [Part 2]({{< ref "dont_take_my_word_for_it_2" >}}) we will use a trick to work out mystery types using compiler error messages.