[Polymorphic Cloning <<](./problem_31.md) | [**Home**](../README.md) | [>> Generalize the Visitor Pattern! Part 2!](./problem_33.md)

# Problem 36: Logging
## **2025-11-26**

We want to encapsulate logging functionality and "add" it to any class.

CRTP approach:

```C++
template<typename T, typename Data> class Logger {
public:
    void loggedSet(Data x) {
        std::cout << "Setting data to " << x << std::endl;
        static_cast<T*>(this)->set(x);  // No virtual call overhead
    }
};

class Box: public Logger<Box, int> {
    friend class Logger<Box, int>;
    
    int x;
    void set(int y) { x = y; }
public:
    Box(): x{0} { loggedSet(0); }
};

Box b;
b.loggedSet(100);
// etc.
```

Another approach:

```C++
class Box {
    int x;
public:
    Box(): x{0} {}
    void set(int y) { x = y;}
};

// Mixin Inheritance
template<typename T, typename Data> class Logger : public T {
public:
    void loggedSet(Data x) {
        std::cout << "Setting data to " << x << std::endl;
        set(x); // No vtable overhead
    } 
};

using BoxLogger = Logger<Box, int>;
BoxLogger b;
b.loggedSet(1);
b.loggedSet(4);
//etc.
```
- One way is logger on top, and the other is underneath
- What to choose? Up to personal preference 

**Mixin inheritance** - can mix and match subclass functionality without writing new subclasses, just write new "decorating" classes and inherit from them

Note: if `SpecialBox` is a subclass of `Box`, then `SpecialBox` has no relation to `Logger<Box, int>`, nor is there any relationship between `Logger<SpecialBox, int>`, `Logger<Box, int>`.

With the CRTP solution, `SpecialBox` is a subtype of `Logger<Box, int>`
- Can specialize behaviour of virtual functions.

---
[Polymorphic Cloning <<](./problem_31.md) | [**Home**](../README.md) | [>> Generalize the Visitor Pattern! Part 2!](./problem_33.md)
