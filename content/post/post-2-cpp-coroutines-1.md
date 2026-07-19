---
title:       "Notes on C++ Coroutines - 1"
subtitle:    ""
description: ""
date:        2026-07-17T11:50:00+08:00
author:      ""
image:       ""
tags:        ["c++", "coroutines"]
categories:  ["Tech" ]
---

# Background
[TODO]

# Core Constructs in C++ Coroutines
A valid coroutine function is a function that must satisfy the following core requirements necessarily (thought not efficiently):
* at least one of `co_await`, `co_return`, `co_yield` statements is present.
* the returned type must be a type containing a specific type `promise_type`.

Below is a typical coroutine function `createSeqGenerator` that works as an even integer sequence geneator:

```cpp
class SeqGenerator {
public:
    class promise_type {
        ...
    };

    int next();
    std::coroutine_handle<promise_type> handle;
};

SeqGenerator createSeqGenerator(int n) {
    for (int i = 0; i < n; i++) {
        co_await (2*i);
    }
}

int main() {
    int n = 5;
    SeqGenerator gen = createSeqGenerator(n);
    for (int i = 0; i < n; i++) {
        int v = gen.next();
        std::cout << v << ", ";
    }
    std::cout << std::endl;
}
```

An object of type `SeqGenerator` represents an active coroutine. This object acts as a channel between the coroutine itself and its caller, as shown later. More importantly, this coroutine object is 1-1 associated with an object of the nested promise type.

 The coroutine will normally pause or suspend at each `co_await`/`co_yield` statement and return an even integer to its caller `main()`. (`co_yield` is a syntactic sugar of `co_await`, so we will use only `co_await` in the following). Next call to this coroutine, here `gen.next()`, will resume this coroutine from previous `co_await` statement and suspend again at the next `co_await` statement. We call `co_await` statements `suspension points` in coroutines. 

In summary, a valid coroutine is a function that contains at least one suspension point and returns a specific type. The following sections will explain how the suspension points and the coroutine objects are constructed and how they work cooperatively together.

## 1 Awaiter type
We know that a coroutine must have at least one `co_await expr` statement that defines a suspension point. This statement actually trigger multiple calls on an so-called `awaiter` object. `Awaiter` type must implement certain interfaces, similarly as `promise_type`, including `await_ready`, `await_suspend`, `await_resume`.

### `bool await_ready()`
This function returns a bool and is called when the coroutine reaches this suspension point. `true` is returned when this coroutine should not be suspended and should continue execution without pause, while `false` if it's needed be suspended right here. Usually we need suspend it here.

### `await_suspend(std::coroutine_handle<> handle)`
If it's determined that the coroutine will be suspended, then this function will be called. The `handle` argument is an access to the associated promise object. It allows to access or modify the coroutine's current state, which will be explained below.

This function is allow to be defined with multiple different return types for different purposes:

* `void`: specifies that the next step is to suspend here and return control to the caller
* `bool`:
    - `true`: same as `void`.
    - `false`: resumes this coroutine immediately
* `std::coroutine_handle<>`: if returning a handle of other routines, it means the current coroutine is handing over control to another routine, i.e., another routine is resumed.

### `await_resume`
This function is called when the current coroutine is resumed by e.g. its caller from this suspension point. Its return type can be any type, and is exactly the `co_await` statement's return type.

C++ standard provides two common-used awaiter types `std::suspend_always` and `std::suspend_never`. The below is a typical implementation of `suspend_always` which will suspend coroutines always.

```cpp
class suspend_always {
public:
    bool await_ready() noexcept { return false; }
    void await_suspend(std::coroutine_handle<>) noexcept { return; }
    void await_resume() noexcept { return; }
};
```

In summary, an anwaiter type defines what the current coroutine would do at this suspension point.

## 2 Coroutine returned type and `promise_type`

It's mentioned above that an object of coroutine returned type represents an active coroutine, and this object is the only channel that allows callers to interact with the coroutine. Importantly, a coroutine object is 1-1 associated with a promise object of the nested `promise_type`. At the moment, we just need know that this promise object is designed to keep the coroutine context in heap when it's suspended, so that it can be resumed from where it's suspended. Usually we need define a coroutine handle in the coroutine returned type that points to the associated promise object, so that a coroutine object is able to access its current context or state.

A valid coroutine returned type must contain a nested type `promise_type` and this promoise type must implement certain interfaces as below. The common used interfaces include `get_return_object`, `initial_suspend`, `final_suspend`, `unhandled_exception`, `await_transform` and a few others.

### `get_return_object`
This function is required, and will be called to construct a coroutine object. Most time we just need a typical implementation as:

```cpp
SeqGenerator get_return_object() {
    return SeqGenerator(std::coroutine_handle<promise_type>::from_promise(*this));
}
```

This implementation just creates a coroutine handle from its promise object `this` and then contructs and returns the coroutine object. This funcion is called when users call a coroutine function. Note that the promise object is constructed by compiler implicitly, meaning that programmers worry about only construction of coroutine objects and not about their associated promise objects. 

### `initial_suspend` and `final_suspend`
The two are required. The `initial_suspend` is called at the begining when the coroutine starts running, while `final_suspend` is called at the end when this coroutine finishes. They both return an awaiter object that define two suspension points respectively.

### `unhandled_exception`
This is required. It's about exception handling and we ignore it for this article.

### `await_transform`
This is not required. It's designed to implicitly convert the `expr` following the keyword `co_await` to an awaiter object. For example, `co_await 2*i` will construct an awaiter object via `await_transform(int)`. The parameter signature corresponds to `expr`. Based on this, we can safely infer that it's allowed to define mutiple overloaded `await_transform` with different parameter signatures to handle different `co_await` statements if needed.

The below are example implementations of the key 3 interfaces:

```cpp
std::suspend_always initial_suspend() { return {}; }

std::suspend_always final_suspend() noexcept { return {}; }

std::suspend_always await_transform(int i) {
    cur_val = i;
    return {};
}
```

As said above, it's not necessary to define a `await_transform` in some cases. This happens when all `co_await` is applied on a concrete awaiter object directly, or when no implicit conversion is needed.

### Constructors of `promise_type`
Promise type's constructors should correspond to the parameter signature of the coroutine function. For example, given `SeqGenerator createSeqGenerator(int n)`, a constructor `promise_type(int)` must be present.

### Coroutine returned type
This type itself is exposed to callers to interact with coroutines. It's free to define various functions, e.g., `SeqGenerator::next()`, that can fetch coroutines' state now or instruct what they should do next. Usually this type will contain a `std::coroutine_handle` that provides functionalities to access and control coroutines, e.g., `resume()` is to resume coroutines.

# Example
Here I give a viable coroutine implementation that demonstrate all the concepts above. Particulary, I give a customized awaiter type `my_suspend_always`.

```cpp
#include <iostream>
#include <thread>
#include <string>

#include <coroutine>

class my_suspend_always {
public:
    bool await_ready() noexcept {
        std::cout << "thread: " << std::this_thread::get_id() << ", my_suspend_always::await_ready returns false" << std::endl;
        return false;
    }

    void await_suspend(std::coroutine_handle<> handle) noexcept {
        (void)handle;
        std::cout << "thread: " << std::this_thread::get_id() << ", my_suspend_always::await_suspend returns void" << std::endl;
        return;
    }

    void await_resume() noexcept {
        std::cout << "thread: " << std::this_thread::get_id() << ", my_suspend_always::await_resume returns void" << std::endl;
        return;
    }
};

class SeqGenerator {
public:
    class promise_type {
    public:
        SeqGenerator get_return_object() {
            return SeqGenerator(std::coroutine_handle<promise_type>::from_promise(*this));
        }

        promise_type() {
            std::cout << "thread: " << std::this_thread::get_id() << ", promise_type::promise_type()" << std::endl;
        }

        promise_type(int n): num(n), cur_val(-1) {
            std::cout << "thread: " << std::this_thread::get_id() << ", promise_type::promise_type(int), num = " << num << ", cur_val = " << cur_val << std::endl;
        }

        my_suspend_always initial_suspend() {
            std::cout << "thread: " << std::this_thread::get_id() << ", promise_type::initial_suspend() returns suspend_always, num = " << num << ", cur_val = " << cur_val << std::endl;
            return {};
        }

        my_suspend_always final_suspend() noexcept {
            std::cout << "thread: " << std::this_thread::get_id() << ", promise_type::final_suspend() returns suspend_always." << std::endl;
            return {};
        }

        void unhandled_exception() {}

        my_suspend_always await_transform(int i) {
            std::cout << "thread: " << std::this_thread::get_id() << ", promise_type::await_transform(int i): i = " << i << std::endl;
            cur_val = i;
            return {};
        }

    private:
        friend class SeqGenerator;
        int num;
        int cur_val;
    };

    SeqGenerator(std::coroutine_handle<promise_type> h): handle(h) {}

    int next() {
        std::cout << "thread: " << std::this_thread::get_id() << ", SeqGenerator::next()" << std::endl;
        handle.resume();
        return handle.promise().cur_val;
    }

private:
    std::coroutine_handle<promise_type> handle;
};

SeqGenerator createSeqGenerator(int n) {
    for (int i = 0; i < n; i++) {
        co_await (2*i);
    }
}

int main() {
    int n = 5;
    SeqGenerator gen = createSeqGenerator(n);
    for (int i = 0; i < n; i++) {
        int v = gen.next();
        std::cout << v << ", ";
    }
    std::cout << std::endl;
}
```

An example output is like

```shell
user@workstation:~/coroutine$ ./generator 2> /dev/null
thread: 136886885377920, promise_type::promise_type(int), num = 5, cur_val = -1
thread: 136886885377920, promise_type::initial_suspend() returns suspend_always, num = 5, cur_val = -1
thread: 136886885377920, my_suspend_always::await_ready returns false
thread: 136886885377920, my_suspend_always::await_suspend returns void
thread: 136886885377920, SeqGenerator::next()
thread: 136886885377920, my_suspend_always::await_resume returns void
thread: 136886885377920, promise_type::await_transform(int i): i = 0
thread: 136886885377920, my_suspend_always::await_ready returns false
thread: 136886885377920, my_suspend_always::await_suspend returns void
thread: 136886885377920, SeqGenerator::next()
thread: 136886885377920, my_suspend_always::await_resume returns void
thread: 136886885377920, promise_type::await_transform(int i): i = 2
thread: 136886885377920, my_suspend_always::await_ready returns false
thread: 136886885377920, my_suspend_always::await_suspend returns void
thread: 136886885377920, SeqGenerator::next()
thread: 136886885377920, my_suspend_always::await_resume returns void
thread: 136886885377920, promise_type::await_transform(int i): i = 4
thread: 136886885377920, my_suspend_always::await_ready returns false
thread: 136886885377920, my_suspend_always::await_suspend returns void
thread: 136886885377920, SeqGenerator::next()
thread: 136886885377920, my_suspend_always::await_resume returns void
thread: 136886885377920, promise_type::await_transform(int i): i = 6
thread: 136886885377920, my_suspend_always::await_ready returns false
thread: 136886885377920, my_suspend_always::await_suspend returns void
thread: 136886885377920, SeqGenerator::next()
thread: 136886885377920, my_suspend_always::await_resume returns void
thread: 136886885377920, promise_type::await_transform(int i): i = 8
thread: 136886885377920, my_suspend_always::await_ready returns false
thread: 136886885377920, my_suspend_always::await_suspend returns void
user@workstation:~/coroutine$
user@workstation:~/coroutine$ ./generator 1> /dev/null
0, 2, 4, 6, 8, 
user@workstation:~/coroutine$
```