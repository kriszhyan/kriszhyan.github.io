---
category: c++
permlink: /:category/:title
---

Let us talk about the perfect forwarding problem. Consider the following code.

```C++
void func(int& a) { cout << "called lvalue ref version" << endl; }
void func(int&& a) { cout << "called rvalue ref version" << endl; }

template <typename T> void wrapper(T&& x) {
  // calling func(x)
}

int main() {
  int a = 2;
  wrapper(a);    // a is an lvalue
  wrapper(2);    // 2 is an rvalue
  // ...
```

To perfectly forward the arguments to `func`, the pre-C++11 approach was to
overload `wrapper` for both `T&` and `const T&`. It has apparent drawbacks,
including the exponential growth of overloads with the number of template
parameters.

C++11 introduces move semantics and special template parameter deduction rules
to solve the perfect forwarding problem. The notion `T&&`, named forwarding
reference, is treated specially during template parameter deduction. Two rules
apply.

1. Reference collapsing, which basically says `T& &&`, `T&& &` and `T& &` all
collapse to `T&`, while `T&& &&` to `T&&`;
2. Type deduction for forwarding reference. For the shown template, if `a` is an
lvalue, `T` is deduced to `T&`; otherwise, `T` to `T`.

Combining these two rules, we see that `wrapper(a)` deduces to `void
wrapper(int& x)`, and `wrapper(2)` yields `void wrapper(int&& x)`. However, C++
considers named variables as lvalues, regardless to their original value
category. Hence, inside `wrapper`, `x` is in fact treated as an lvalue.

To forward the value category of `x`, the standard way is to use `std::forward`
(or a similar cast), and write the `func` call as
`func(std::forward<T>(x))`. `std::forward` is implemented as follows (in C++11).

```C++
template <typename T> T&& forward(typename std::remove_reference<T>::type& arg) {
  return static_cast<T&&>(arg);
}
```

Now let us verify that it indeed works. Since `a` is an lvalue, `wrapper(a)`
yields `void wrapper(int& x)` with `T = int&`. Thus `func(std::forward<T>(x))`
becomes `func(std::forward<int&>(x))`, which collapses to

```C++
int& forward(int& arg) { return static_cast<int&>(arg); }
```

On the other hand, `wrapper(2)` yields `void wrapper(int&& x)` with `T =
int`. Thus `func(std::forward<T>(x))` becomes `func(std::forward<int>(x))`,
which collapses to

```C++
int&& forward(int& arg) { return static_cast<int&&>(arg); }
```

Apparently, `std::forward` does give what we want.

If we dive further, a few interesting questions arise.

1. Can we discard `std::remove_reference` and just write `T&& forward(T& arg)`?
2. How about `T&& foward(T&& arg)`?

Well, for the first question, the answer is in some sense, we can. It is easy to
verify that without `std::remove_reference`, we are still able to perfectly
forward arguments. But, it only works if we specify the type parameter, i.e.,
`forward<T>(x)` instead of `forward(x)`. The reason is that, if we do not fully
specify the type parameters, type deduction will be triggered. As `x` is treated
as an lvalue in `wrapper`, `forward` deduces that `T = int`. Note that special
rules are not used because `T&` is not a forwarding reference. Thus `func`
always receives an `int&&`. That is to say, the sole purpose of
`std::remove_reference` is to force the programmer to specify the type
parameter.

The second question can be reasoned similarly. If we omit the type parameter,
`forward` deduces `T = int&` due to the lvalue-ness of `x`. Note that special
rules are induced here. No matter what value category `x` was, `func` always
receives an `int&`. On the other hand, if we do specify the type parameter as
`forward<T>(x)`, we get compilation error due to `x` being an lvalue and
`forward` expecting an rvalue.

In C++14, there is another overload of `std::forward` for rvalues. Most of the
time, the lvalue version is used. Only when return values need to be forwarded,
the rvalue version is called. For example,
`func(std::forward(return_int(x)))`. Details of template parameter deduction
rules can be found
[here](http://en.cppreference.com/w/cpp/language/template_argument_deduction).