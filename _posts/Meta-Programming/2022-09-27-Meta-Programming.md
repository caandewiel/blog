---
layout: post
title: Template Metaprogramming in C++
---

Template metaprogramming is a form of metaprogramming. In a nutshell, metaprogramming is the technique of programming where the application is capable of using another program as its data (read and/or write) or even modifying itself.

## Iterating over `std::tuple`
The standard library does not add any functionality for looping over the values in `std::tuple`. The main reason for this, is that `std::tuple` is a variadic template. This means that it has an arbitrary number of template parameters. This differs for instance from a `std::vector`, which *always* has just one template parameter.

```cpp
std::tuple<int, float> a = {1, 2.0f};

// This would be problematic for the compiler, as an iterator is required, but
// `el` can be a different type every iteration.
for (auto el : a) {
  // Do something with `el`.
}
```

So how do we fix this? Well, a very simple solution would be:

```cpp
template <typename T> void tuple_for_each(T t) {
  for (size_t I = 0; I < std::tuple_size<T>(); ++I) {
    auto &el = std::get<I>(t);
    // Do something with `el`.
  }
}
```

Or is it? Actually, no, it is not! See how the index `I` in `std::get<I>` is a template parameter? This means that `I` should be known during compile time and currently we are determining `I` during runtime.

## Iterating during compile time
The solution here is template metaprogramming! Using template metaprogramming we can loop over specific datatypes during compile time whereas a generic for-loop in C++ will only function during runtime.

I hear you ask, if I cannot use a for-loop to iterate, how am I going to iterate over the data structure then? Well, that's where recursion comes into play. Let's use the following setup that allows us to loop over the values in a tuple and prints them.

```cpp
template <typename T> void print_something(T t) {
  std::cout << typeid(t).name() << " - " << t << std::endl;
}

int main() {
  tuple_for_each(std::tuple<int, float>(),
                 [](auto &&x) { print_something(x); });

  return 0;
}
```

Let's focus on the `tuple_for_each` function. This will be a templated function defining two different types, `...T` corresponding to the parameter pack of our tuple and `F` corresponding to the lambda function. The signature of this function would look like this:

```cpp
template <typename F, typename... T>
constexpr void tuple_for_each(std::tuple<T...> &&t, F &&f)
```

In order to iterate over the tuple, we should keep track of an index at compile time. We can do this, by adding an integer as a template parameter.

```cpp
template <int I, typename F, typename... T>
constexpr void tuple_for_each(std::tuple<T...> &&t, F &&f)
```

Our recursive template will look like:

```cpp
template <typename F, typename... T>
constexpr void tuple_for_each(std::tuple<T...> &&t, F &&f) {
  tuple_for_each<0, F, T...>(std::forward<decltype(t)>(t), std::forward<F>(f));
}

template <int I, typename F, typename... T>
constexpr void tuple_for_each(std::tuple<T...> &&t, F &&f) {
  if
    constexpr(I < sizeof...(T)) {
      f(std::get<I>(t));
      tuple_for_each<I + 1, F, T...>(std::forward<decltype(t)>(t),
                                     std::forward<F>(f));
    }
}
```
Important to note here is that we need to define a base case during compile time. We can achieve this by using a compile time conditional, namely `if constexpr(I < sizeof...(T))`. When index `I` is less than the size of the tuple, we call `tuple_for_each` with `I + 1`. The base case will then automatically be achieved when `I >= sizeof...(T)` as the function will basically return without calling itself again.

The complete code snippet is defined as follows:
```cpp
template <typename T> void print_something(T t) {
  std::cout << typeid(t).name() << " - " << t << std::endl;
}

template <typename F, typename... T>
constexpr void tuple_for_each(std::tuple<T...> &&t, F &&f) {
  tuple_for_each<0, F, T...>(std::forward<decltype(t)>(t), std::forward<F>(f));
}

template <size_t I, typename F, typename... T>
constexpr void tuple_for_each(std::tuple<T...> &&t, F &&f) {
  if
    constexpr(I < sizeof...(T)) {
      f(std::get<I>(t));
      tuple_for_each<I + 1, F, T...>(std::forward<decltype(t)>(t),
                                     std::forward<F>(f));
    }
}

int main() {
  tuple_for_each(std::tuple<int, float>(),
                 [](auto &&x) { print_something(x); });

  return 0;
}
```
Executing this yields the following result. We can see that the tuple is properly being iterated over.
```console
foo@bar:~$ ./example
i - 0
f - 0
```
