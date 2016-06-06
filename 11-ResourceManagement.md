# Resource Management 
This section contains rules related to resources. A resource is anything that must be acquired and (explicitly or implicitly) released, such as memory, file handles, sockets, and locks. The reason it must be released is typically that it can be in short supply, so even delayed release may do harm. The fundamental aim is to ensure that we don't leak any resources and that we don't hold a resource longer than we need to. An entity that is responsible for releasing a resource is called an owner.

There are a few cases where leaks can be acceptable or even optimal: If you are writing a program that simply produces an output based on an input and the amount of memory needed is proportional to the size of the input, the optimal strategy (for performance and ease of programming) is sometimes simply never to delete anything. If you have enough memory to handle your largest input, leak away, but be sure to give a good error message if you are wrong. Here, we ignore such cases.

## Manage resources automatically using resource handles and RAII (Resource Acquisition Is Initialization)

To avoid leaks and the complexity of manual resource management. C++'s language-enforced constructor/destructor symmetry mirrors the symmetry inherent in resource acquire/release function pairs such as fopen/fclose, lock/unlock, and new/delete. Whenever you deal with a resource that needs paired acquire/release function calls, encapsulate that resource in an object that enforces pairing for you -- acquire the resource in its constructor, and release it in its destructor.

Example, bad

Consider:
```cpp
void send(X* x, cstring_span destination)
{
    auto port = OpenPort(destination);
    my_mutex.lock();
    // ...
    Send(port, x);
    // ...
    my_mutex.unlock();
    ClosePort(port);
    delete x;
}
```
In this code, you have to remember to unlock, ClosePort, and delete on all paths, and do each exactly once. Further, if any of the code marked ... throws an exception, then x is leaked and my_mutex remains locked.

For example, consider:
```cpp
void send(std::unique_ptr<X> x, cstring_span destination)  // x owns the X
{
    Port port{destination};            // port owns the PortHandle
    lock_guard<mutex> guard{my_mutex}; // guard owns the lock
    // ...
    Send(port, x);
    // ...
} // automatically unlocks my_mutex and deletes the pointer in x
```

Now all resource cleanup is automatic, performed once on all paths whether or not there is an exception. As a bonus, the function now advertises that it takes over ownership of the pointer. But, what is Port? A handy wrapper that encapsulates the resource:
```cpp
class Port {
    PortHandle port;
public:
    Port(cstring_span destination) : port{OpenPort(destination)} { }
    ~Port() { ClosePort(port); }
    operator PortHandle() { return port; }

    // port handles can't usually be cloned, so disable copying and assignment if necessary
    Port(const Port&) =delete;
    Port& operator=(const Port&) =delete;
};
```
### Note:
Where a resource is "ill-behaved" in that it isn't represented as a class with a destructor, wrap it in a class or use `finally`.

### Summary

* Use constructor/destructor pairs to make sure anything that was allocated and initialized is also desalloacated/uninitialized as necessary
* Eliminates the possibility that an unexpected code path could prevent cleanup
* Can actually be more efficient



## Interfaces

An interface is a contract between two parts of a program. Precisely stating what is expected of a supplier of a service and a user of that service is essential. Having good (easy-to-understand, encouraging efficient use, not error-prone, supporting testing, etc.) interfaces is probably the most important single aspect of code organization.

## Pointers in Interface
* Pointers in interfaces:
  * Should only be used to represent a single item
  * Shoud only be used to represent a single `optional` item 
* Consider using Guideline Support Library `span` class
* Consider using `optional` for optional parameters
* Use the tools available: C++ core checker


