
# ASYNC : :


Notes: 
- it's up to the C++ implementation (like GCC or Clang) to decide whether to execute the function on a separate thread or not. It's not guaranteed to always create a new thread.

--- 

- `std::async` is used to execute functions asynchronously.
The function template `std::async` runs the function f asynchronously (potentially in a separate thread which might be a part of a thread pool) and returns a [std::future](https://en.cppreference.com/w/cpp/thread/future "cpp/thread/future") that will eventually hold the result of that function call.

The launch policies (`std::launch::async` and `std::launch::deferred`) determine when and how the function is executed.

- `std::launch::async` ensures immediate execution of the function.
- `std::launch::deferred` defers execution until the result is requested.
- The default launch policy allows the implementation to choose between immediate execution and deferral.
<br>
1.
 `auto a1 = std::async(&X::foo, &x, 42, "Hello");`
-> This line asynchronously calls the member function `foo` of object `x` with arguments `42` and `"Hello"`. The default launch policy is used, which means the function may execute concurrently or be deferred until `a1.get()` is called.
<br>
 2.
 `auto a2 = std::async(std::launch::deferred, &X::bar, x, "world!");`
 -> This line asynchronously calls the member function `bar` of object `x` with the argument `"world!"`, but with a deferred launch policy. This means the function will be deferred until `a2.get()` or `a2.wait()` is called.
<br>
3.
`auto a3 = std::async(std::launch::async, X(), 43);`
-> This line asynchronously creates a temporary object of type `X` and calls its `operator()` with argument `43`. The function is executed with the async launch policy, meaning it will execute concurrently.



```
std::async(std::launch::async , []() {

});

or

std::async(std::launch::async ,function F , arg1 , arg2 ...);

```

- as it launches the process into an separate thread so , it be like execute that asynchronously and  pass that to the Future so that it process be get  in future object and we can later get the result when we wanted or when it is finished calculating the result . 

- ==`[std::future]`referring to the shared state created by this call to `std::async`.==

##### Parameters  :

- f              - 	Callable object to call
- args 	- 	parameters to pass to f
- policy 	- 	bit-mask value, where individual bits control the  allowed methods of execution 

### Exceptions : 

- std::bad_alloc : memory for the internal data structure cannot be allocated
- std::system_error : if not able to start a new thread

### Important : 

If the [std::future](https://en.cppreference.com/w/cpp/thread/future "cpp/thread/future") obtained from `std::async` is not moved from or bound to a reference, the destructor of the [std::future] will block at the end of the full expression until the asynchronous operation completes, essentially making code such as the following synchronous:
```cpp
std::async([std::launch::async], []{ f(); }); // temporary's dtor waits for f()
std::async([std::launch::async], []{ g(); }); // does not start until f() completes
```

Note that the destructors of [std::future] is obtained by means other than a call to `std::async` never block.

Example code :

```cpp
#include <algorithm>
#include <future>
#include <iostream>
#include <mutex>
#include <numeric>
#include <string>
#include <vector>
 
std::mutex m;
 
struct X
{
    void foo(int i, const std::string& str)
    {
        std::lock_guard<std::mutex> lk(m);
        std::cout << str << ' ' << i << '\n';
    }
 
    void bar(const std::string& str)
    {
        std::lock_guard<std::mutex> lk(m);
        std::cout << str << '\n';
    }
 
    int operator()(int i)
    {
        std::lock_guard<std::mutex> lk(m);
        std::cout << i << '\n';
        return i + 10;
    }
};
 
template<typename RandomIt>
int parallel_sum(RandomIt beg, RandomIt end)
{
    auto len = end - beg;
    if (len < 1000)
        return std::accumulate(beg, end, 0);
 
    RandomIt mid = beg + len / 2;
    auto handle = std::async(std::launch::async,
                             parallel_sum<RandomIt>, mid, end);
    int sum = parallel_sum(beg, mid);
    return sum + handle.get();
}
 
int main()
{
    std::vector<int> v(10000, 1);
    std::cout << "The sum is " << parallel_sum(v.begin(), v.end()) << '\n';
 
    X x;
    // Calls (&x)->foo(42, "Hello") with default policy:
    // may print "Hello 42" concurrently or defer execution
    auto a1 = std::async(&X::foo, &x, 42, "Hello");
    // Calls x.bar("world!") with deferred policy
    // prints "world!" when a2.get() or a2.wait() is called
    auto a2 = std::async(std::launch::deferred, &X::bar, x, "world!");
    // Calls X()(43); with async policy
    // prints "43" concurrently
    auto a3 = std::async(std::launch::async, X(), 43);
    a2.wait();                     // prints "world!"
    std::cout << a3.get() << '\n'; // prints "53"
} // if a1 is not done at this point, destructor of a1 prints "Hello 42" here
```
---


# FUTURE :

#### When : 

if you have server logic that needs to access an API on another server, you don't want to block your application waiting on the answer. Using `std::future` (or the corresponding asynchronous concept in another programming language) seems to be the correct answer.

On the other hand, you don't always know which local functions are going to access a remote API , so you might need to have every function and method returning an `std::future`. Futures might even leak to function parameters and data members (so in the end everything becomes a future).

you can call a remote API in async code, then pass the result of that call into sync code for the bulk of your processing.

#### About : 
A promise is an object that can store a value of type T to be retrieved by a [future](https://cplusplus.com/future) object (possibly in another thread), offering a synchronization point.


A `std::future<T>` is a handle to a result of work which is [potentially] not, yet, computed.

You can imagine it as the receipt you get when you ask for work and the receipt is used to get the result back.


```cpp
std::future<int> f = std::async(std::launch::async, []{
```cpp
    // long calculation
    return /* some result */;
});
/* do some other stuff */
int result = f.get();
```

`std::async` with the `std::launch::async` flag runs a function (here a lambda) asynchronously (in another thread). It returns a `std::future`, which will _eventually_, when that function finishes, contain an `int` value.

`std::future` is used only in multithreaded programs. Such programs use threads, which can be thought of as mutually independent subprograms, to perform some tasks

---
##### Problem :

these "threads", "workers", "subprograms" need to communicate to share / transfer some data securely. The problem is that since they are completely independent, one needs a mechanism with which the "producer" of the information can either send it to the "consumer" or communicate its failure to produce the data; similarly, the "consumer" or "receiver" must have a secure, reliable way to tell whether a) the data from another thread is ready b) receive the data and c) receive the error signal should the "producer" fail to produce the data.

##### Solution : 

One method of providing such synchronization mechanism is a **promise-future** pair.
`std::promise` is used by the producer to set (and "send") the data, whereas `std::future` is used by "consumer" to receive it.

If the producer cannot fulfil the contract, it may satisfy it with an exception ,which is a C++ method of telling that something has gone wrong. Then, the "consumer" will receive this exception instead of data via its `std::future`.

---

### Summary : 

`std::future` is an object used in multi-threaded programming to receive data or an exception from a different thread; it is one end of a single-use, one-way communication channel between two threads, `std::promise` object being the other end. Using the promise-future pattern, you have a guarantee that your inter-thread communication is free of common errors specific to multi-threaded programs, like race conditions or accessing already freed memory, which are very common if the threads "talk to each other" using methods designed for single-threaded programs, like when function (=thread) arguments are being passed by reference.

##### Members functions : 

| [operator=]  | moves the future object  <br>(public member function)                                                                                                        |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [share]      | transfers the shared state from *this to a [shared_future](https://en.cppreference.com/w/cpp/thread/shared_future "cpp/thread/shared future") and returns it |
| [get]        | returns the result                                                                                                                                           |
| [valid]      | checks if the future has a shared state  <br>(public member function)                                                                                        |
| [wait]       | waits for the result to become available  <br>(public member function)                                                                                       |
| [wait_for]   | waits for the result, returns if it is not available for the specified timeout duration  <br>(public member function)                                        |
| [wait_until] | waits for the result, returns if it is not available until specified time point has been reached  <br>(public member function)                               |

---
# OPTIONAL :

std::optional powerful feature with flexibility and improved error handling .
It's a template class that represent an optional value , meaning it can either hold a value or no value at all .

Especially useful in scenarios where a function any not always return a valid result . 

its mainly memory efficient , zero runtime overhead , achieves this by using a small amount of internal storage to represent the absence of a value, resulting in efficient memory usage and performance.
##### Syntax : 

- for empty optional 
	`std::optional<int> emptyOptional;`
	 // The constructor can be called with no arguments to create an empty std::optional object.Alternative can use with values to assign it 

```cpp
destinationOptional = optionalWithValue;  
// Assign value from one std::optional to anotherstd::optional<int> emptyDestinationOptional;  
emptyDestinationOptional = emptyOptional;  // Assign empty std::optional to another std::optional
```

#### For Accessing the value : 

1. value() function // if not any value then can cause undefined behavior
2. value_or(default_value) function // access the underlying value or use default value if empty
3. has_value()  function --> accessing the underlying value only if it is present

#### Deleting And Replacing Value : 
```cpp
std::optional<int> myOptional;  
myOptional = 42;
// Modify the value using emplace()  
myOptional.emplace(100);// Clear the value using reset()  
myOptional.reset();
```


### Important  :
- Use of the std::optional is with std::promise like when you want to represent the possibility of having a value in the future, but you're not sure if the value will be available.

```cpp
    std::optional<int> result = future_result.get();

    if (result.has_value()) {
        std::cout << "Result: " << result.value() << std::endl;
    } else {
        std::cout << "Result is not available" << std::endl;
    }
```

- also can use `std::nullopt` to detect the null values 
```cpp
auto create2(bool b)
{
    return b ? std::optional<[std::string]>{"Godzilla"} : [std::nullopt]
}
```


---

# Promise :

The class template `std::promise` provides a facility to store a value or an exception that is later acquired asynchronously via a [std::future] object created by the `std::promise` object. Note that the `std::promise` object is meant to be used only once.


###### The `std::promise` and `std::future` implement one-shot producer-consumer semantics.

The consumer (`std::future`) can block until the result of the producer (`std::promise`) is available,

##### Main Points : 

-  With `std::promise`, the result is typically generated synchronously within another function or thread, and then communicated to the waiting thread. The waiting thread blocks until the result is ready.

##### Real Life Example : 

With `std::promise`, think of it like you're making a promise to yourself or someone else that you'll find an answer to a question. Then, you go and find the answer, and when you're done, you fulfill the promise by sharing the answer. The person who was waiting for the answer gets it when you fulfill your promise.




Member Functions : 

| [operator=]                    | assigns the shared state  <br>(public member function)                                                                |
| ------------------------------ | --------------------------------------------------------------------------------------------------------------------- |
| [swap]                         | swaps two promise objects  <br>(public member function)                                                               |
| Getting the result             |                                                                                                                       |
| [get_future]                   | returns a [future] associated with the promised result  <br>(public member function)                                  |
| Setting the result             |                                                                                                                       |
| [set_value]                    | sets the result to specific value  <br>(public member function)                                                       |
| [set_value_at_thread_exit]     | sets the result to specific value while delivering the notification only at thread exit  <br>(public member function) |
| [set_exception]                | sets the result to indicate an exception  <br>(public member function)                                                |
| [set_exception_at_thread_exit] | sets the result to indicate an exception while delivering the notification only at thread exit                        |

```cpp
void accumulate(std::vector<int>::iterator first,
                std::vector<int>::iterator last,
                std::promise<int> accumulate_promise)
{
    int sum = std::accumulate(first, last, 0);
    accumulate_promise.set_value(sum); // Notify future
}
 
void do_work(std::promise<void> barrier)
{
    std::this_thread::sleep_for(std::chrono::seconds(1));
    barrier.set_value();
}
 
int main()
{
    // Demonstrate using promise<int> to transmit a result between threads.
    std::vector<int> numbers = {1, 2, 3, 4, 5, 6};
    std::promise<int> accumulate_promise;
    std::future<int> accumulate_future = accumulate_promise.get_future();
    std::thread work_thread(accumulate, numbers.begin(), numbers.end(),std::move(accumulate_promise));
 
    // future::get() will wait until the future has a valid result and retrieves it.
    // Calling wait() before get() is not needed
    // accumulate_future.wait(); // wait for result
    std::cout << "result=" << accumulate_future.get() << '\n';
    work_thread.join(); // wait for thread completion
 
    // Demonstrate using promise<void> to signal state between threads.
    std::promise<void> barrier;
    std::future<void> barrier_future = barrier.get_future();
    std::thread new_work_thread(do_work, std::move(barrier));
    barrier_future.wait();
    new_work_thread.join();
}
```


#### States : 
1. `_make ready_` : the promise stores the result or the exception in the shared state.Marks the state ready and unblocks any thread waiting on a future associated with the shared state.
2. `_release_`  : the promise gives up its reference to the shared state. If this was the last such reference, the shared state is destroyed. Unless this was a shared state created by [std::async] which is not yet ready, this operation does not block.
3. _abandon_: the promise stores the exception of type [std::future_error] with error code [std::future_errc::broken_promise], makes the shared state _ready_, and then _releases_ it.


`std::promise` is another way to communicate the result of asynchronous operations between threads. It allows one thread (the producer) to set a value or an exception, which can be retrieved by another thread (the consumer) through a corresponding `std::future`. `std::promise` is particularly useful when you want to provide a value asynchronously and have another thread wait for that value to become available.


A promise must either be satisfied via `set_value()` or have an exception set via `set_exception()` before its lifetime ends if its future is to be consumed. A satisfied promise can die without consequence, and `get()` becomes available on the future. A promise with an exception will raise the stored exception upon call of `get()` on the future. If the promise dies with neither value nor exception, calling `get()` on the future will raise a "broken promise" exception.

#### Points :

1. Represents a placeholder for a value or an exception to be set in the future.
2. Associated with a `std::future`, which will hold the result or exception.
3. Doesn't directly execute a task; instead, it's used to set a value or an exception.
4. Typically involves synchronous generation of the result within another function or thread.
5. The waiting thread blocks until the result is set using the promise.



**Case 1 : Exception **

```cpp
int test()
{
    std::promise<int> pr;
    auto fut = pr.get_future();

    {
        std::promise<int> pr2(std::move(pr));
        pr2.set_exception(std::make_exception_ptr(std::runtime_error("Booboo")));
    }

    return fut.get();
}
// throws the runtime_error exception
```

**Case 2: Broken promise**

```cpp
int test()
{
    std::promise<int> pr;
    auto fut = pr.get_future();

    {
        std::promise<int> pr2(std::move(pr));
    }   // Error: "broken promise"

    return fut.get();
}
```

---

## Packaged task : 

It's useful when you want more control over when and where the function runs, especially when you want to explicitly specify the thread for execution.

we may insist that the function be executed in a separate thread. We already know that we can provide a separate thread by means of the [`std::thread`](http://en.cppreference.com/w/cpp/thread/thread) class.

The next lower level of abstraction does exactly that: [`std::packaged_task`](http://en.cppreference.com/w/cpp/thread/packaged_task). This is a template that wraps a function and provides a future for the functions return value, but the object itself is call­able, and calling it is at the user's discretion

```cpp
std::packaged_task<int(double, char, bool)> tsk(foo);

auto fut = tsk.get_future();    // is a std::future<int>
```


### Definition Acc . to reference : 

The class template `std::packaged_task` wraps any [Callable]  target (function, lambda expression, bind expression, or another function object) so that it can be invoked asynchronously. Its return value or exception thrown is stored in a shared state which can be accessed through [std::future] objects.

###### Notes : 
- Just like [std::function](https://en.cppreference.com/w/cpp/utility/functional/function "cpp/utility/functional/function"), `std::packaged_task` is a polymorphic, allocator-aware container: the stored callable target may be allocated on heap or with a provided allocator.

#### Main Point :

With `std::packaged_task`, the execution of the task happens asynchronously on another thread. The main thread can continue with its own work while waiting for the result to be ready.

#### Real Life Example : 

With `std::packaged_task`, imagine you give a task (like a math problem) to a friend and ask them to solve it. They go off, work on it, and bring back the answer to you. You're waiting for them to finish solving the problem.

### Member Functions :

| [operator=]                 | moves the task object                                                                                                                                          |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [valid]                     | checks if the task object has a valid function  <br>(public member function)                                                                                   |
| [swap]                      | swaps two task objects  <br>(public member function)                                                                                                           |
| [get_future]                | returns a [std::future](https://en.cppreference.com/w/cpp/thread/future "cpp/thread/future") associated with the promised result  <br>(public member function) |
| [operator()]                | executes the function  <br>(public member function)                                                                                                            |
| [make_ready_at_thread_exit] | executes the function ensuring that the result is ready only once the current thread exits  <br>(public member function)                                       |
| [reset]                     | resets the state abandoning any stored results of previous executions  <br>(public member function)                                                            |
| [std::swap]                 | specializes the [std::swap](https://en.cppreference.com/w/cpp/algorithm/swap "cpp/algorithm/swap") algorithm                                                   |


```cpp
int f(int x, int y) { return std::pow(x, y); }
 
void task_lambda()
{
    std::packaged_task<int(int, int)> task([](int a, int b)
    {
        return std::pow(a, b); 
    });
    std::future<int> result = task.get_future();
 
    task(2, 9);
 
    std::cout << "task_lambda:\t" << result.get() << '\n';
}
 
void task_bind()
{
    std::packaged_task<int()> task(std::bind(f, 2, 11));
    std::future<int> result = task.get_future();
 
    task();
 
    std::cout << "task_bind:\t" << result.get() << '\n';
}
 
void task_thread()
{
    std::packaged_task<int(int, int)> task(f);
    std::future<int> result = task.get_future();
 
    std::thread task_td(std::move(task), 2, 10);
    task_td.join();
 
    std::cout << "task_thread:\t" << result.get() << '\n';
}
 
int main()
{
    task_lambda();
    task_bind();
    task_thread();
}
```


```cpp
int foo(double , char , bool)

auto fut = std::async(foo , 1.5 , false);
auto res = fut.get();

std::packaged_task<int(double,char,bool) > tsk(foo);
tsk.get_future();  // is a std::future<int>
auto res = fut.get(); // as before



now to implement the packaged_task this ie where the promise comes in 

The Promise is the building block of for communication with a future. 

```



## Points :

1. Wraps a callable object (like a function or a lambda) to be executed asynchronously.
2. Associates the callable object with a `std::future`, which represents the result of the task.
3. Allows passing arguments to the callable object when creating the `std::packaged_task`.
4. Execution of the task happens asynchronously on another thread.
5. The main thread can continue its work while waiting for the result to be ready.


---
## Note : 


Promise ->  not copyable , only movable
packaged_task -> not copyable or movable
std::async -> as return future so not copyable but movable
promise -> not copyable , nor movable
optional -> copyable and also copy the content it holds 


---
