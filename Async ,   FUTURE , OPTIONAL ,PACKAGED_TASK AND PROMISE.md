
# ASYNC : :
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

- as it launches the process into an seprate thread so , it be like execute that asynchronously and  pass that to the Future so that it process be get  in future object and we can later get the result when we wanted or when it is finished calculating the result . 

- ==[std::future](https://en.cppreference.com/w/cpp/thread/future "cpp/thread/future") referring to the shared state created by this call to `std::async`.==


##### Parameters  :

- f 	- 	Callable object to call
- args 	- 	parameters to pass to f
- policy 	- 	bitmask value, where individual bits control the  allowed methods of 
				execution 

---


# FUTURE :

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




---

# Promise :







---

## Packaged task : 

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


