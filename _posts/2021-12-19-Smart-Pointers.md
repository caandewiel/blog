---
layout: post
title: Smart Pointers in C++
---

Smart pointers are nothing more than a simple wrapper around regular pointer types in C++, which ensure that an object is automatically deleted when it’s not being used anymore. Long story short, they are an extremely powerful tool to prevent memory leaks.

Let’s take a very simple C++ structure to demonstrate the power of smart pointers.

```cpp
struct Object {
  Object() { std::cout << "Creating new Object\n"; }
  ~Object() { std::cout << "Destroying Object\n"; }

  uint64_t someAttribute;
};
```

In old school C++ we would do a heap allocation using the `new` keyword. This will conveniently allocate some memory on the heap for us and return a pointer to the object. However, C++ does require you to *explicitly* call the destructor when the object is not used anymore. The keyword to do this is `delete`, this is similar to `malloc` and `free` respectively. If you fail to do this, you introduce a so called memory leak. This is the situation where memory on the heap is allocated with data that is not being used anymore. Imagine that you allocate these objects iteratively. After a while you will run out of memory! This situation is described by the following code snippet.

```cpp
void doSomething() {
  // Allocate an instance of Object on the heap
  auto* object = new Object();
}

int main() {
  doSomething();

  // Memory is not freed here, thus causing a memory leak
  return 0;
}
```

Running this piece of code results in the following output. We can see here that only the constructor is called, meaning that the object is still alive after we get out of scope of the main function.

```shell
> Creating new Object
```

Since C++ 11, we have smart pointers to take care of freeing up memory for us! When it comes to smart pointers, it’s intuitive to think of ownerships. But what does it mean for an instance to *own* another instance? Say we have an instance `a` which is the owner of instance `b`. Then instance `b` can never outlive instance `a`. In other words, when instance `a` is destroyed, instance `b` is destroyed as well.

## Unique pointers

The first smart pointer we will cover is a unique pointer, defined by `std::unique_ptr<T>`. A unique pointer is, as the name describes, a pointer that can only be owned by one instance. In order to visualize this a bit more, let’s define two structs `A` and `B`.

```cpp
struct A {
  A() { std::cout << "Creating new A\n"; }
  ~A() { std::cout << "Destroying A\n"; }

  uint64_t someAttribute;
};

struct B {
  std::unique_ptr<A> a;
};

int main() {
  B b;
  b.a = std::unique_ptr<A>(new A());
}
```

Here instance `b` of type `B` has an attribute `a` of type `std::unique_ptr<A>`. This means that `b` is the owner of `a`. Since `b` is allocated on the stack, it will automatically be destroyed when it gets out of the current scope. All smart pointers do, is destroy the object corresponding to the pointer when it has no owners anymore. Since `a` is a unique pointer, it can only have **one** owner, which in this case is `b`. This means that when the destructor of `b` is called, the destructor of `a` will be called automatically. When we run this piece of code, we get the following output.

```shell
> Creating new A
> Destroying A
```

We can see that *without* calling the `delete` function on `a`, the destructor is still called. This is extremely useful when your codebase becomes larger and it gets harder to keep track of all pointers you have floating around.

## So how does this work internally?

It might sound complicated, but actually, it is a very simple yet powerful concept. Here is an extremely simplified implementation of the unique pointer.

```cpp
template <typename T>
class unique_ptr {
 public:
  unique_ptr(T* t) : m_pointer(t) {}
  ~unique_ptr() { delete m_pointer; }

 private:
  T* m_pointer;
};
```

We can immediately spot one serious vulnerability here. You can actually instantiate a unique pointer on the heap as well (e.g. `new std::unique_ptr<A>(new A())`). You **never** want to do this! This will prevent the internal pointer from actually being freed as its destructor will not automatically be called when it gets out of scope. Remember to **always** allocate smart pointers on the stack!

## Can’t we just copy smart pointers?

Yes and no! Well, actually no. Theoretically it is possible, but C++ explicitly prevents us from doing that. Let’s first analyze why we do not want to do this. We can actually demonstrate this with the rough implementation we wrote before.

```cpp
auto a = unique_ptr<A>(new A());
auto b = a;
```

We copy the value of `a` here into a variable `b`. This program compiles just fine, but during runtime we spot something interesting! The output of the aforementioned program is.

```shell
> Creating new A
> Destroying A
> Destroying A
> free(): double free detected in tcache 2
```

We see here that our program attempts to free the same pointer twice, why is that? Well, we have two unique pointers pointing to the same address. However, the destructor of `A` is called twice, once by `a` and once by `b`. Therefore our program attempts to free the pointer twice. So how do we avoid this problem? Well, it’s a simple fix! We just prevent our unique pointers from being copied.

```cpp
template <typename T>
class unique_ptr {
 public:
  unique_ptr(T* t) : m_pointer(t) {}
  ~unique_ptr() { delete m_pointer; }

  // Remove the copy constructor and operator
  unique_ptr(const unique_ptr<T>&) = delete;
  unique_ptr<T>& operator=(const unique_ptr<T>&) = delete;

 private:
  T* m_pointer;
};
```

When we try to compile our program with our updated unique pointer, we get the following compilation error telling us that we are trying to call a deleted function, which in this case is the copy constructor.

```bash
> error: use of deleted function ‘unique_ptr::unique_ptr(const unique_ptr&) [with T = A]’
  auto b = a;
```

I hear you ask, so how do I change ownership if I cannot copy the unique pointer? Well, C++ also has a very convenient function to overcome that problem, namely `std::move()`. This function basically casts your lvalue to an rvalue, which will thus use the move constructor `unique_ptr(const unique_ptr&&)` instead of our deleted copy constructor. In order to add support for this, we need to add the move constructor.

```cpp
template <typename T>
class unique_ptr {
 public:
  unique_ptr(T* t) : m_pointer(t) {}
  ~unique_ptr() { delete m_pointer; }

  // Remove the copy constructor and operator
  unique_ptr(const unique_ptr<T>&) = delete;
  unique_ptr<T>& operator=(const unique_ptr<T>&) = delete;

  // Define the move constructor and operator
  unique_ptr(unique_ptr<T>&& other) {
    m_pointer = other.m_pointer;
    other.m_pointer = nullptr;
  }
  unique_ptr<T>& operator=(unique_ptr<T>&& other) {
    m_pointer = other.m_pointer;
    other.m_pointer = nullptr;
    return this;
  }

 private:
  T* m_pointer;
};
```

Now we can change our code to **move** the unique pointer instead of copy, which runs fine without freeing the same address twice.

```cpp
auto a = unique_ptr<A>(new A());
auto b = std::move(a);
```

## Shared pointers

Next to unique pointers, there are also shared pointers. They are almost identical to unique pointers, but there is one important difference! As the name describes here as well, shared pointers can have **multiple** owners. The difference implementation wise is that shared pointers **can** be copied. However, in order to know when we need to free our internal pointer, we should keep track of how many active owners we have. We can do this by creating a counter we share between our different shared pointers. Once we create a new shared pointer, we increase this counter and vice versa. Once this counter hits 0, we free both the internal pointer and the counter.

```cpp
struct counter {
  uint64_t count = 1;
};

template <typename T>
class shared_ptr {
 public:
  shared_ptr(T* t) : m_pointer(t) {
    m_counter = new counter();
    std::cout << "Created shared pointer\n";
    std::cout << "Shared pointer has " << m_counter->count
              << " active owners\n";
  }
  ~shared_ptr() {
    std::cout << "Destructor of shared pointer was called.\n";
    m_counter->count--;
    std::cout << "Shared pointer has " << m_counter->count
              << " active owners\n";

    if (m_counter->count == 0) {
      std::cout << "Freeing shared pointer.\n";
      delete m_pointer;
      delete m_counter;
    }
  }

  T* get() { return m_pointer; }

  // Define the copy constructor and operator
  shared_ptr(shared_ptr<T>& other) {
    m_pointer = other.m_pointer;
    other.m_counter->count++;
    m_counter = other.m_counter;
    std::cout << "Shared pointer has " << m_counter->count
              << " active owners\n";
  }
  shared_ptr<T>& operator=(shared_ptr<T>& other) {
    m_pointer = other.m_pointer;
    other.m_counter->count++;
    m_counter = other.m_counter;
    std::cout << "Shared pointer has " << m_counter->count
              << " active owners\n";
    return *this;
  }

  // Define the move constructor and operator
  shared_ptr(shared_ptr<T>&& other) {
    m_pointer = other.m_pointer;
    other.m_pointer = nullptr;
  }

  shared_ptr<T>& operator=(shared_ptr<T>&& other) {
    m_pointer = other.m_pointer;
    other.m_pointer = nullptr;
    return *this;
  }

 private:
  T* m_pointer;
  counter* m_counter;
};
```

And when we run the following code we get this output.

```cpp
auto a = shared_ptr<A>(new A());
auto b = a;
```

```shell
> Creating new A
> Created shared pointer
> Shared pointer has 1 active owners
> Shared pointer has 2 active owners
> Destructor of shared pointer was called.
> Shared pointer has 1 active owners
> Destructor of shared pointer was called.
> Shared pointer has 0 active owners
> Freeing shared pointer.
> Destroying A
```

## Weak pointers

The weak pointer is the last type of smart pointer we find in C++. A weak pointer is in fact just a shared pointer which does not increase the internal counter. Weak pointers have many different use cases, but let’s take cyclic dependency as an example to demonstrate how weak pointers work. Let’s take the classic example of a doubly linked list here. This means that each element in the list should have the pointer for the next element, but also for the previous element. 

```cpp
struct Node {
  Node() { std::cout << "Creating new Node\n"; }
  ~Node() { std::cout << "Destroying Node\n"; }

  shared_ptr<Node> next;
  shared_ptr<Node> previous;
};

auto initial = shared_ptr<Node>(new Node());
initial.get()->next = shared_ptr<Node>(new Node());
initial.get()->next.get()->previous = initial;
```

This creates two nodes, but none of these get destroyed as their cyclic dependency keeps the counter greater than 0. When we run the aforementioned code, we get the following output.

```shell
> Creating new Node
> Creating new Node
```

Let’s rewrite our implementation to use weak pointers. The implementation for weak pointers is almost identical to that one of shared pointers, but without the counter logic. We end up with the following implementation.

```cpp
// Declare weak_ptr as friend class of shared_ptr, so we can read private
// members.
template <typename T>
class weak_ptr {
 public:
  weak_ptr() = default;
  weak_ptr(shared_ptr<T> other) {
    m_pointer = other.m_pointer;
    m_counter = other.m_counter;
  }

  ~weak_ptr() = default;

  T* get() { return m_pointer; }

  // Define the copy constructor and operator
  weak_ptr(weak_ptr<T>& other) {
    m_pointer = other.m_pointer;
    m_counter = other.m_counter;
  }
  weak_ptr<T>& operator=(weak_ptr<T>& other) {
    m_pointer = other.m_pointer;
    m_counter = other.m_counter;
    return *this;
  }

  // Define the move constructor and operator
  weak_ptr(weak_ptr<T>&& other) {
    m_pointer = other.m_pointer;
    other.m_pointer = nullptr;
  }

  weak_ptr<T>& operator=(weak_ptr<T>&& other) {
    m_pointer = other.m_pointer;
    other.m_pointer = nullptr;
    return *this;
  }

 private:
  T* m_pointer;
  counter* m_counter;
};

struct Node {
  Node() { std::cout << "Creating new Node\n"; }
  ~Node() { std::cout << "Destroying Node\n"; }

  shared_ptr<Node> next;
  weak_ptr<Node> previous;
};

int main() {
  auto initial = shared_ptr<Node>(new Node());
  auto next = shared_ptr<Node>(new Node());
  initial.get()->next = next;
  initial.get()->next.get()->previous = initial;
}
```

This gives us the desired result.

```shell
> Creating new Node
> Creating new Node
> Destroying Node
> Destroying Node
```

## Conclusion

In this overview we have analyzed the three different smart pointers in C++, namely unique, shared and weak pointers. We showed how smart pointers are an extremely helpful tool in preventing memory leaks. Last but not least, we implemented our own simplified versions of all three smart pointers to get a grasp of how they work internally and what kind of overhead you can expect.

**Happy (memory leak free) coding!**