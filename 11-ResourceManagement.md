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
##### Note:
Where a resource is "ill-behaved" in that it isn't represented as a class with a destructor, wrap it in a class or use `finally`.

##### Summary about RAII

* Use constructor/destructor pairs to make sure anything that was allocated and initialized is also desalloacated/uninitialized as necessary
* Eliminates the possibility that an unexpected code path could prevent cleanup
* Can actually be more efficient

## In interfaces, use raw pointers to denote individual objects (only)

An interface is a contract between two parts of a program. Precisely stating what is expected of a supplier of a service and a user of that service is essential. Having good (easy-to-understand, encouraging efficient use, not error-prone, supporting testing, etc.) interfaces is probably the most important single aspect of code organization.

In interfaces, use raw pointers to denote individual objects (only). Why? 
Because arrays are best represented by a container type (e.g., vector (owning)) or a span (non-owning). Such containers and views hold sufficient information to do range checking.

Example, bad ideia
```cpp
void f(int* p, int n)   // n is the number of elements in p[]
{
    // ...
    p[2] = 7;   // bad: subscript raw pointer. Bang!
    // ...
}
```
The compiler does not read comments, and without reading other code you do not know whether p really points to n elements. So, we can use a span instead.

Example
```cpp
void g(int* p, int fmt)   // print *p using format #fmt
{
    // ... uses *p and p[0] only ...
}
```
Exception: C-style strings are passed as single pointers to a zero-terminated sequence of characters. Use zstring rather than char* to indicate that you rely on that convention.

##### Note

Many current uses of pointers to a single element could be references. However, where nullptr is a possible value, a reference may not be an reasonable alternative.

##### Enforcement

* Flag pointer arithmetic (including ++) on a pointer that is not part of a container, view, or iterator. This rule would generate a huge number of false positives if applied to an older code base.
* Flag array names passed as simple pointers


##### Summary about pointers in interface

* Pointers in interfaces:
  * Should only be used to represent a single item
  * Shoud only be used to represent a single `optional` item 
* Consider using Guideline Support Library `span` class
* Consider using `optional` for optional parameters
* Use the tools available: C++ core checker

## Avoid defining any default operations, or define them all

If you can avoid defining default operations, do. The reason for that 
it's the simplest and gives the cleanest semantics. For example:

```cpp
struct NamedMap {
public:
    // ... no default operations declared ...
private:
    std::string name;
    std::map<int, int> rep;
};

NamedMap nm;        // default construct
NamedMap nm2 {nm};  // copy construct
```
Since std::map and std::string have all the special functions, no further work is needed.

But, if you define or =delete any default operation, define or =delete them all, because the semantics of the special functions are closely related, so if one needs to be non-default, the odds are that others need modification too.
Example of a bad idea
```cpp
struct M2 {   // bad: incomplete set of default operations
public:
    // ...
    // ... no copy or move operations ...
    ~M2() { delete[] rep; }
private:
    std::pair<int, int>* rep;  // zero-terminated set of pairs
};

void use()
{
    M2 x;
    M2 y;
    // ...
    x = y;   // the default assignment: double free......
    // ...
}
```
Given that "special attention" was needed for the destructor (here, to deallocate), the likelihood that copy and move assignment (both will implicitly destroy an object) are correct is low (here, we would get double deletion).

#####  Note

* The rule for define all of them is known as "the rule of five" or "the rule of six", depending on whether you count the default constructor. And the rule's name for define none of them is rule of zero.
* If you want a default implementation of a default operation (while defining another), write =default to show you're doing so intentionally for that function. If you don't want a default operation, suppress it with =delete.
* Compilers enforce much of this rule and ideally warn about any violation.
* Relying on an implicitly generated copy operation in a class with a destructor is deprecated.

##### Enforcement

(Simple) A class should have a declaration (even a =delete one) for either all or none of the special functions

* If you must manage a resource, you must have constructor/destructors and utilize RAII
* If you manage a resource, define, =default or =delete all of the special functions
* If you manage a resource, manage exactly one resource.
 

#####  Enforcement

(Not enforceable) While not enforceable, a good static analyzer can detect patterns that indicate a possible improvement to meet this rule. For example, a class with a (pointer, size) pair of member and a destructor that deletes the pointer could probably be converted to a vector.


##### Summary for Rule of 0

* Nerver define any of the special functions
* With class initializers this can even include the default constructor
* Likely will result in less code and smaller/more efficient compiled code

# Prefer Stack Objects - Don't heap-allocate unnecessarily

A scoped object is a local object, a global object, or a member. This implies that there is no separate allocation and deallocation cost in excess of that already used for the containing scope or object. The members of a scoped object are themselves scoped and the scoped object's constructor and destructor manage the members' lifetimes.
For Example, the following example is inefficient (because it has unnecessary allocation and deallocation), vulnerable to exception throws and returns in the ... part (leading to leaks), and verbose:

```cpp
void f(int n)
{
    auto p = new Gadget{n};
    // ...
    delete p;
}
```
Instead, use a local variable:
```cpp
void f(int n)
{
    Gadget g{n};
    // ...
}
```
#####  Enforcement

* (Moderate) Warn if an object is allocated and then deallocated on all paths within a function. Suggest it should be a local auto stack object instead.
* (Simple) Warn if a local Unique_ptr or Shared_ptr is not moved, copied, reassigned or reset before its lifetime ends.

