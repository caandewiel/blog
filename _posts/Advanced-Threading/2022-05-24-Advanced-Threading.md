---
layout: post
title: Multithreading in C++
---

According to [cplusplus](https://www.cplusplus.com/reference/thread/thread/), a thread is *a sequence of instructions that can be executed concurrently with other such sequences in multithreading environments, while sharing a same address space.* But what does this mean and why would we want to use this?

## What is threading?
Let's zoom in on the definition of a thread. We can see a thread as a sequence of instructions which can be controlled independently. All sequences can, because they are independent, be ran concurrently as long as the CPU allows for this. Important to note is that multithreading is **not** the same as multiprocessing and the answer lies in the difference between threads and processes. A process is the execution of a complete sequence representing a program while a thread, as mentioned before, is just an independent sequence of instructions. We can see threads as a subset of a process, in other words, a process can launch new threads. Because threads are managed by the same process, they share an identical address space which is not the case for processes. This is a very powerful, yet vulnerable feature when not properly used. The handling of threads is managed by the operating system and luckily something that we as high level C++ developers do not need to worry about!

## How to use `std::thread`?
Threads in C++ are represented by the class `std::thread` and can be accessed through the `#include <thread>` header. In order to properly compile a program using `std::thread`, the compiler flag `-lpthread` must be used. So how do we use `std::thread` in a real program?

```cpp
#include <cstdio>
#include <thread>

int main() {
  std::thread t([]() { printf("Do some random stuff here\n"); });
  t.join();

  return 0;
}
```
___
```console
foo@bar:~$ ./example
Do some random stuff here
```

We can provide a lambda function to the constructor of `std::thread`. After the construction of `t`, the thread will be picked up by the OS thread scheduler as soon as possible. The function `join()` is a blocking function that waits for the thread to finish execution. Not calling `join()` before exiting the program can cause a segmentation fault.

If we simplify the process of spawning and joining threads in C++, we can represent schematically as seen in the image below. Here we create two seperate threads, `t_0` and `t_1`, which get picked up by the OS thread scheduler and assigned to specific cores. This doesn't mean that this code doesn't work on a system with only one physical core, it just means it won't be able to run concurrently.

{% include_relative /threads.drawio.svg %}

## Race conditions
Let's build on top of the example from the previous section by extending it with an extra thread. 

```cpp
#include <cstdio>
#include <thread>

int main() {
  std::thread t_1([]() { printf("Do some random stuff here from thread 1\n"); });
  std::thread t_2([]() { printf("Do some random stuff here from thread 2\n"); });
  t.join();

  return 0;
}
```
When running this program several times, you'll notice that sometimes `t_1` finishes before `t_2` and the other way around.
```console
foo@bar:~$ ./example
Do some random stuff here from thread 1
Do some random stuff here from thread 2
```
```console
foo@bar:~$ ./example
Do some random stuff here from thread 2
Do some random stuff here from thread 1
```
There is no way to say beforehand which thread finishes first. So any of the aforementioned outputs could be valid outputs of the program. This fully depends on the scheduling algorithm of the operating system and the current workload.

For this example, there is not a very big problem, as both threads are just printing some text. But we have learned that threads have a shared address space. This means that they can access **and** alter the same pieces of memory at the same time when not properly handled. The situation where multiple threads are "racing" for access to the same memory address is called a race condition.

An example of a race condition can be found in the following code snippet. Basically, we have one static variable `value`, which both threads want to alter. Since both threads are spawned before any of them are joined, thread `t_1` could be executed before `t_2` and vice versa. Both threads are thus *racing* for access to `value`.

```cpp
#include <cstdio>
#include <thread>

static int value;

int main() {
  std::thread t_1([]() { value = 1; });
  std::thread t_2([]() { value = 2; });
  t_1.join();
  t_2.join();

  printf("%d\n", value);

  return 0;
}
```

## Mutual exclusion using `std::mutex`
Mutual exclusion is a so called synchronization primitive that can be used to prevent race conditions and is implementented in the standard library as the class `std::mutex`.

A mutex (mutual exclusion) can be seen as a public toilet. The first person to get to the toilet, occupies it and locks the door. This way no one else can get in. After this person is done, he/she will unlock the door and leave the toilet. Now the second person can go in. If we stay with the toilet example, we can easily understand what the different functions in `std::mutex` do.

|Method|Function|Toilet Equivalent|
|:-|:-|:-|
|`.lock()`|Locks the mutex|Lock the toilet door|
|`.try_lock()`|Tries to lock the mutex. Returns `true` when the lock was acquired and `false` otherwise.|Try to occupy the toilet and locking the door, but there might be someone on the toilet at this time|
|`.unlock()`|Unlocks the mutex|Unlock the toilet door|

So now let's apply a mutex to our previous race condition. First of all we have to create a static mutex as it will be shared between both threads. Instead of providing the thread constructor a lambda function, we provide it the function and its arguments. In the function `alterValue` we lock the mutex before altering the value. This means that whichever thread comes first, gets to lock the mutex first and thus blocking the other thread. Once this thread is done, it will unlock the thread again so it can be picked up by the next thread. This way two threads can never alter the same data at the same time.

```cpp
#include <cstdio>
#include <mutex>
#include <thread>

static int value;
static std::mutex mutex_value;

void alterValue(int i) {
  mutex_value.lock();
  value = i;
  mutex_value.unlock();
}

int main() {
  std::thread t_1(alterValue, 1);
  std::thread t_2(alterValue, 2);
  t_1.join();
  t_2.join();

  printf("%d\n", value);

  return 0;
}
```
C++ comes with a nice templated class called `std::lock_guard`. This class tries to gain ownership over the provided mutex and automatically releases it once it gets out of scope. We can rewrite the `alterValue` function using a lock guard as follows.
```cpp
void alterValue(int i) {
  std::lock_guard<std::mutex> lock(mutex_value);
  value = i;
}
```
At the end of the function, `lock` will go out of scope and automatically be destructed and thus freeing `mutex_value`.