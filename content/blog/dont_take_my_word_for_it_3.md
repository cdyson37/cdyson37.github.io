---
title: "Don't Take My Word For It (Part 3): A handy reference table"
date: 2021-12-27
summary: "How are template arguments to a function deduced? We will generate a helpful reference table."
---

In Part 1 we looked at the `typeid` operator and in Part 2 we used a little cheat to work out mystery types using compiler error messages.

Let's take this a bit further and try to answer a general question: if I call some function template `f(T x)`, `f(T &x)`, `f(T &&x)` (or `const` versions of any of those) with an `int`, `int&` or `int&&` (or `const` versions of any of those), to what does `T` deduce? And what's the resulting type of `x`?

That's 36 possible combinations in each of two tables (one for `T` and one for `x`). We can work them out on an individual basis with the technique from Part 2. For example if we wanted to know what happens if we call `f(T&& x)` and `x` is `int&&`:

```c++
#include <utility>

template <typename T>
struct Incomplete;

template <typename T>
void f(T&& x)
{
    (void) x; // suppress compiler warning
    Incomplete<decltype (x)> incomplete;
}

int main ()
{
    int some_int = 0;
    f(std::move (some_int));
}
```

Output (if you've got a friendly-ish compiler):
```
prog.cc: In instantiation of 'void f(T&&) [with T = int]':
prog.cc:16:6:   required from here
prog.cc:10:30: error: 'Incomplete<int&&> incomplete' has incomplete type
   10 |     Incomplete<decltype (x)> incomplete;
      |                              ^~~~~~~~~~
```

The compiler tells us two things here, courtesy of the helpful error message:
* `T` is `int`, and
* `x` is `int&&`.

This is exactly what we'd hope to see - `f()` gets an rvalue reference to a moved-from variable.

One case down then, but there are 35 to go to fill out our 6x6 table. Let's try to do them all in one go and make a snazzy table!

Firstly, let's cheat a little and encode the possible outcome types as strings. We know it can only be `int`, `int&` etc. so there aren't too many:
```c++
template <typename T>
constexpr char* TypeName;

template <>
constexpr char TypeName<int>[] = "int";

template <>
constexpr char TypeName<int&>[] = "int&";

template <>
constexpr char TypeName<int&&>[] = "int&&";

template <>
constexpr char TypeName<const int>[] = "const int";

template <>
constexpr char TypeName<const int&>[] = "const int&";

template <>
constexpr char TypeName<const int&&>[] = "const int&&";
```
These are [variable templates](https://en.cppreference.com/w/cpp/language/variable_template) introduced in C++14. So `TypeName<int>` will just give us a C-string of `"int"`. If we've missed out a `TypeName` specialisation, we will get a compiler error as the base template for `TypeName` is not initialised.

Next up, for convenience's sake we make a little pair type to store the deduced `T` and the type of `x` in each case. We could work out `x` ourselves but again, let's make the compiler do the work and reduce the risk of a mistake. The name of the struct will make sense in a moment:

```c++
template <typename T, typename U>
struct ReturnedType
{
  using deduced_template_argument = T; // type of T
  using argument_type = U;             // type of x
};
```

So - we want to try out e.g. `template <typename T> f(T x)` with `x` as `int`, `int&` etc. But it's hard to talk about `f` since it's a template and not a function. We can't just take its address or use it as an argument to a template without first selecting a `T`, which totally undermines what we're trying to do since we're trying to find out what `T` would deduce to!

The solution is to wrap our `f` in something that we can give a name to:
```c++
struct CalledByValue
{
  template <typename T>
  static
  auto f (T x) -> ReturnedType<T, decltype (x)>;
};
```

`CalledByValue` is an ordinary struct, not a template, and we can use it as a template argument. Rather than make the inner `f` just return void, we've used our `ReturnedType` to encode both `T` and the type of `x`. We never actually call `f` (we're just going to ask the compiler what type it would return), so there's no need to actually provide a definition for the function.

Next we also define `CalledByReference`:
```c++
struct CalledByReference
{
  template <typename T>
  static
  auto f (T& x) -> ReturnedType<T, decltype (x)>;
};
```
i.e. exactly the same but with an extra `&`. Similarly we then define `CalledByForwardingReference` (takes `T&&`), `CalledByConstValue` (takes `const T`), `CalledByConstReference` (takes `const T&`), `CalledByConstForwardingReference` (takes `const T&&`).

We're almost there. We just need a little helper that takes e.g. `CalledByValue` and `int&` as template arguments and returns either a string of type `T` or a string of the type of `x`. Firstly we'll define an enum to choose which of those two things we want in our table:
```c++
enum class TableType
{
  DEDUCED_TEMPLATE_ARGUMENT,
  ARGUMENT_TYPE
};
```

Our function is then
```c++
template <typename Target, typename Arg, TableType table_type>
auto get_type_name ()
{

  if constexpr (!requires {Target::f (std::declval<Arg> ());})
  {
    return "-";
  }
  else
  {
    using Result = decltype (Target::f (std::declval<Arg> ()));

    if constexpr (table_type == TableType::ARGUMENT_TYPE)
    {
      return TypeName<typename Result::argument_type>;
    }
    else
    {
      return TypeName<typename Result::deduced_template_argument>;
    }
  }
}
```
So `Target` will be something like `CalledByValue` and `Arg` will be something like `int&`.

Our first `if` just returns a `"-"` if it's impossible to call `Target::f` with an argument of type `Arg`. The `requires` keyword gives us a bool that can be used in a `constexpr` context. "if constexpr requires" doesn't sound terribly grammatical to me, but it gets the job done.

You might be tempted instead to just try `if constexpr (!std::is_invocable<decltype (Target::f), Arg> ())`, but that won't work because `Target::f` doesn't have a type - it's a template! If we don't have C++20 and `requires` to hand then the problem can be solved with some [void_t](https://en.cppreference.com/w/cpp/types/void_t) trickery.

Once we're past that first `if constexpr` the rest falls into place nicely. We know it's safe to do the `decltype` because we've established that `Target::f` really can be called with an `Arg`. We know `Result` is then `ReturnedType<T, decltype(x)>` (because that's what all the `f`s return) so it's then a matter of extracting one of those two things from `ReturnedType` and converting that to a string with `TypeName`.

The rest of the program is just handle-turning. We could try to be clever and iterate through the various types with variadics but I think that just makes it harder to read.

First, a function to make to make a single table row. Here `table_type` specifies whether we want `T` or `decltype(x)` and `Target` as before might be `CalledByValue`.
```c++
template <TableType table_type, typename Target>
void make_table_row ()
{
  std::cout << get_type_name<Target, int, table_type> () << '|'
            << get_type_name<Target, int&, table_type> () << '|'
            << get_type_name<Target, int&&, table_type> () << '|'
            << get_type_name<Target, const int, table_type> () << '|'
            << get_type_name<Target, const int&, table_type> () << '|'
            << get_type_name<Target, const int&&, table_type> () << "|\n";
}
```

We use the pipe symbol as a delimiter so I can easily put this into a Markdown table.

Making a whole table just means calling this once per row, with a header at the top:
```c++
template <TableType table_type>
void make_table ()
{
  std::cout << "||int|int&|int&&|const int|const int &|const int &&|\n";
  std::cout << "|-|-|-|-|-|-|-|\n";

  std::cout << "|f(T x)|";
  make_table_row<table_type, CalledByValue> ();

  std::cout << "|f(const T x)|";
  make_table_row<table_type, CalledByConstValue> ();

  std::cout << "|f(T& x)|";
  make_table_row<table_type, CalledByReference> ();

  std::cout << "|f(const T& x)|";
  make_table_row<table_type, CalledByConstReference> ();

  std::cout << "|f(const T&& x)|";
  make_table_row<table_type, CalledByConstForwardingReference> ();

  std::cout << "|f(T&& x)|";
  make_table_row<table_type, CalledByForwardingReference> ();
}
```

Finally `main()` just makes the two types of table:
```c++
int main ()
{
  std::cout << "T deduces as ... when called with:\n";
  make_table<TableType::DEDUCED_TEMPLATE_ARGUMENT> ();

  std::cout << '\n';
  std::cout << "decltype(x) is ... when called with:\n";
  make_table<TableType::ARGUMENT_TYPE> ();
}
```

Phew. Here's the complete program on [compiler explorer](https://godbolt.org/z/vsTeE3xc1). Now let's look at the two tables:

**T deduces as ... when called with:**
||int|int&|int&&|const int|const int &|const int &&|
|-|-|-|-|-|-|-|
|f(T x)|int|int|int|int|int|int|
|f(const T x)|int|int|int|int|int|int|
|f(T& x)|-|int|-|const int|const int|const int|
|f(const T& x)|int|int|int|int|int|int|
|f(const T&& x)|int|-|int|int|-|int|
|f(T&& x)|int|int&|int|const int|const int&|const int|

**decltype(x) is ... when called with:**
||int|int&|int&&|const int|const int &|const int &&|
|-|-|-|-|-|-|-|
|f(T x)|int|int|int|int|int|int|
|f(const T x)|const int|const int|const int|const int|const int|const int|
|f(T& x)|-|int&|-|const int&|const int&|const int&|
|f(const T& x)|const int&|const int&|const int&|const int&|const int&|const int&|
|f(const T&& x)|const int&&|-|const int&&|const int&&|-|const int&&|
|f(T&& x)|int&&|int&|int&&|const int&&|const int&|const int&&|

To pick an example, if we have `f(T&& x)` and we call it with a `const int `, then `T` will be `const int&` (first table) and `x` will have type `const int&&` (second table). The "-"s represent arguments that can't be used for a particular function - you'd get a compiler error.

The table seems to be consistent with a few half-remembered rules of thumb:
* `T` never deduces as a reference, unless you're in the `f(T&&)` magic case.
* With `f(const T x)`, the `const` makes no difference at all to the deduction of `T`, it's just to prevent us from changing `x` in the function.
* With `f(T& x)`, `T` deduces as `const` whenever it's needed.
* `f(T x)` and `f(const T& x)` always work, no matter what you call them with. In latter case, it will extend a reference to a temporary if necessary.

Another thing to note is that the column for `int` is identical to the column for `int&&`. This makes sense as we never care what sort of rvalue `f()` is being called with: `f(42)` (i.e. with `int`, a prvalue) and `f(std::move(some_int))` (i.e. with `int&&`, an xvalue) should be indistinguishable. We only care if it's an rvalue of one sort or another.

There are four invalid ("-") combinations. In all these cases `T` deduces to `int` (we can confirm by looking at error messages), it's just you can't actually call them:
* `f(T& x)` with `int`. So e.g. we can't call `f(function_returning_3())`. This is because C++ [won't extend the lifetime](https://en.cppreference.com/w/cpp/language/lifetime#Temporary_object_lifetime) of the resulting temporary - it only does that for a const lvalue reference (i.e. `f(const T&)`) or to an rvalue reference (`f(const T&&)` or `f(T&&)`). If this were allowed, and it extended the lifetime of the temporary, it would make it too easy to write code that appears to be modifying a variable passed by reference, but is actually only modifying a temporary.
* `f(T& x)` with `int&&`. In this case there's no need to extend a temporary - our `int` isn't going anywhere. But it's disallowed anyway, presumably for consistency between `int` and `int&&` as discussed above.
* `f(const T&& x)` with `int&`. This is an interesting one. Our `f` in this case does *not* take a forwarding reference. The `const` inhibits the special behaviour, this is back to normal template argument deduction. `T` will never deduce as a reference - it's just plain `int` - so then we'just trying to call `f(int&&)` with an `int&`, which is obviously not allowed (if it were allowed, `f()` might move from something that it's not safe to move from).
* `f(const T&& x)` with `const int&`. Pretty much the same story as above, although because of the `const` it wouldn't actually be dangerous if it were allowed, just pointless and misleading.

# Conclusions

In Part 1 we looked at getting type information dynamically with the `typeid` operator, which produces useful-ish output but is better suited to checking the dynamic type of objects at runtime. In Part 2 we used a trick involving an incomplete type to get the compiler to print the mystery type as an error message. In Part 3 we used variable templates to give strings to the types we were interested in, then produced a table of deduced template arguments when calling templated functions.