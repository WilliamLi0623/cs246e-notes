[Less Copying << ](./problem_14.md) | [**Home**](../README.md) | [>> Is Vector Exception Safe?](./problem_16.md)

# Problem 15: Memory Management is Hard!
## **2025-10-02**

No it isn't!
- Vectors can do everything arrays can
    - Grow as needed O(1) amortized time or better (reserve the space you need).
    - Clean up automatically when they go out of scope.
    - Are tuned to minimize copying.

Just use vectors, and you'll never have to manage arrays again.
C++ has enough abstraction facilities to make programming easier than C.

But what about single objects?

```C++
void f() {
    Posn *p = new Posn{1, 2};
    // ...
    delete p;   // Must deallocate the posn!
}
```

First ask yourself, do you really need to use the heap? Could you have used the stack instead? (remember heap memory allocation is not _that_ time efficient).

```C++
void f() {
    Posn p {1, 2};
    // ...
}   // No cleanup necessary, no need garbo collector, ezpz
```

But sometimes you do need the heap. Calling `delete` isn't so bad, but consider:

```C++
class BadNews {};
void f() {
    Posn *p = new Posn{1, 2};
    if (...) throw BadNews{}; // oops
    delete p;  
}
```

`p` is leaked if `f` throws (unacceptable).

Raising and handling an exception should not corrupt the program. 
- We desire **exception safety**.

Leaks are a corruption of your program's memory. This will eventualy degrade performance and crash the program.

If a program cannot recover from an exception without corrupting its memory, what's the point of recovering?

What constitutes exception safety? 3 levels:
1. **Basic Guarantee** - Once an exception has been handled, the program is in **some** valid state, no leaked memory, no corrupted data structures, all invariants are maintained.
1. **Strong Guarantee** - If an exception propagates out of a function `f`, then the state of the program will be **as if `f` had not been called**.
    - `f` either succeeds completely or not at all, no in between.
1. **Nothrow Guarantee** - A function `f` offers the nothrow guarantee if `f` **never emits an exceptions** and always accomplishes its purpose.


Will revist, but now coming back to `f`:

```C++
void f() {
    Posn *p = new Posn {1, 2};

    if (...) {
        delete p;
        throw BadNews{};
    }
    // ...
    delete p;   // Duplicated effort! Memory management is even harder!
}
```

Want to guarantee that `delete p;` happens no matter what. What guarantee does C++ offer?
- Only one (but this is enough): that destructors for stack-allocated objects will be called when the objects go out of scope.

So create a class with a destructor that deletes the pointer.

```C++
template<typename T> class unique_ptr {
    T *p;
    public:
        unique_ptr(T *p): p{p} {}
        ~unique_ptr() { delete p; }
        T* get() const { return p; }
        T* release() { // give me the pointer then don't have it anymore
            T *q = p;
            p = nullptr;
            return q;
        }
};

void f() {
    unique_ptr<Posn> p {new Posn{1, 2}};
    if (...) throw BadNews{};
}
```

That's it! Less memory management effort than we started with!

Using `unique_ptr` can use `get` to fetch the pointer.

**Better** - Make `unique_ptr` act like a pointer.

```C++
template<typename T> class unique_ptr {
        T *p;
    public:
        ...
        T &operator*() const { return *p; }

        T *operator->() const { return p; }

        explicit operator bool() const { return p; }    // Explicit prohibits bool b = p;
};
```

Overloading `->` is weird.
- On the left is a pointer and on the right is a pointer
- No syntax for specifying something is a field name
- Thus you specify what the pointer is and C++ will automatically grab the field for you

But consider:

```C++
unique_ptr<Posn> p {new Posn{1, 2}}; 
unique_ptr<Posn> q = p;
```

Consider two pointers to the same object! Can't both delete it! (Undefined behaviour)


**Solution:** Copying `unique_ptr`s are not allowed. Moving is ok though.

```C++
template<typename T> class unique_ptr {
    ...
    public:
        unique_ptr(const unique_ptr &other) = delete;
        unique_ptr &operator=(const unique_ptr &other) = delete;
        unique_ptr(unique_ptr &&other): p{other.p} { other.p = nullptr; }
        unique_ptr &operator=(unique_ptr &&other) {
            std::swap(p, other.p);
            return *this;
        }
};
```

**Note:** This is how copying of streams is prevented.

Emplacement for `unique_ptr`: `make_unique`

```C++
template <typename T, typename... Args> unique_ptr<T> make_unique(Args&&... args) {
    return unique_ptr<T> { new T(std::forward<Args>(args)...) };
}
```

Ex.

```C++
auto p = make_unique<Posn>(3,4);
```

`unique_ptr` is an example of the C++ idiom: **Resource Acquisition Is Initialization (RAII)**
- Any resource that must be properly released (memory, file handle, window, etc.) should be wrapped in a stack-allocated object whose destructor frees it.
- Ex. `unique_ptr`, `ifstream`/`ofstream` acquire the resource when the object is initialized and release it when the object's destructor runs.

C++ is one of the languages that don't have a garbage collector, and they don't plan on adding it.
- One of the argument is that garbage collection only solve a part of the problem, it does not solve everything. Yes it cleans up your memory, but you memory is only one of the many things you need to clean up (file, network connection,...). 
- Another thing is that garbage collecting can happen at any time - program must be stopped temporarily, memory must be cleaned, then resume again - which introduces some lags affecting program performance. With dtor, you know exactly when it's gonna happen, it's free and cheap, you don't really need to care about it.

---
[Less Copying << ](./problem_14.md) | [**Home**](../README.md) | [>> Is Vector Exception Safe?](./problem_16.md)