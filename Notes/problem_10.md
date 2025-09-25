[Walk Faster <<](./problem_9.md) | [**Home**](../README.md) | [>> I Want a Vector of Chars](./problem_11.md)

# Problem 10: Now You've Gone Too Far!
## **2025-09-24**

Consider:

```C++
Vector v;
v.push_back(4);
v.push_back(5);

v[2];   // Out of bounds! (undefined behaviour) - may or may not crash
```

Can we make this safer?

Adding bounds checks to operator`[]` may be needlessly expensive.

Could have a second, safer fetch operator (optional for user):

```C++
class Vector {
    ...
    public:
        int &at (size_t i) {
            if (i < n) return theVector[i];
            // else what?
        }
};
```

`v.at(2)` still wrong - what should happen?
- Return any `int`, looks like non-error.
- Returning a non-`int`, not type correct.
- Crash the program - can we do better? Don't return. Don't crash.

**Solution:** Raise an `exception`

```C++
class range_error {};

class Vector {
    ...
    public:
        int &at(size_t i) {
            if (i < n) return theVector[i];
            else throw range_error{};   // Construct an object of range_error & "throw" it
        } 
};
```

## **2025-09-25**

- Client's options
1. Do nothing
    ```C++
    Vector v;
    v.push_back(0);
    v.at(1); // The exception will crash the program
    ```
1. Catch it
    ```C++
    try {
        Vector v;
        v.push_back(0);
        v.at(1);
    } catch (range_error &r) {  // r is the thrown object
        // usually catch by ref, save the cost of copying.
        // Do something
    }
    ```
1. Let your caller catch it
    ```C++
    int f() {
        Vector v;
        v.push_back(0);
        v.at(1);
    }
    int g() {
        try{
            f();
        } catch(range_error &r) {
            // Do something
        }
    }
    ```

    - Exception will propagate through the callchain until a handler is found.
    - Called **unwinding** the stack.
    - If no handler is found, program aborts (`std::terminate` gets called).
    - Control resumes after the catch block (problem code is not retried).

What happens if a constructor throws an exception?
- Object is considered **partially constructed**
- Destructor will not run on partially constructed object

Consider:

```C++
class C {...};

class D {
    C a;
    C b;
    int *c;

    public:
        D() {
            c = new int[10];
            ...
            if (...) throw something {};  // (*)
        }

        ~D() {
            delete[] c;
        }
};

D d;
```

- If `D d;` throws, the `D` object is partially constructed, so `~D()` will not run.
- But `a` and `b` are fully constucted so their destructors will run.
- So if a constructor wants to throw, it must clean itself.

```C++
class D {
    ...
    public:
        D() {
            c = new int[10];
            // ...
            if (...) {
                delete[] c;
                throw something {};
            }
        }
    }
```

What happens if a destructor throws? 
> Trouble
- By default, program aborts immediately (`std::terminate` is called).
- If you really want a throwing destructor, tag it with `noexcept(false)`. 
- But you don't.

```C++
class myexception{};

class C {
    ...
    public:
        ~C() noexcept(false) {
            throw myexception{};
        }
};

void h() {
    C c1;
}

void g() {
    C c2;
    h();
};

void f() {
    try {
        g();
    } catch (myException &) {
        ...
    }
}
```

1. Destructor for `c1` throws at the end of `h`.
1. Unwind through `g`.
1. Destructor for `c2` throws as we leave `g`.
    - No handler yet.
    - Now two unhandled exceptions.
    - Immediate termination guaranteed, `std::terminate` is called.

**Never** let destructors throw, swallow the exception if necessary.

Also note that you can throw _any value_, not just objects.

---
[Walk Faster <<](./problem_9.md) | [**Home**](../README.md) | [>> I Want a Vector of Chars](./problem_11.md)