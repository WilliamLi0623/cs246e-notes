[Heterogeneous Data << ](./problem_21.md) | [**Home**](../README.md) | [>> If a Class Has No Objects](./problem_23.md)

# Problem 22 - I'm Leaking!
## **2025-10-28**

```C++
class X {
        int *a;
    public:
        X(int n): a{new int[n]} {}
        ~X() { delete[] a; }
};

class Y: public X {
        int *b;
    public:
        Y(int n, int m): X{n}, b{new int[m]} {}
        ~Y() { delete[] b; }    // Note: Y's destructor will call X's destructor (step 3)
};

X *px = new Y{3, 4};
delete px;  // Leaks
```

This calls `X`'s destructor, but not `Y`'s
- Now, this problem sounds familiar, when we want to make this pointer pick the correct method based on the actual type, what do we do???
  
**Solution:** make the destructor _virtual_

```C++
class X {
        // ...
    public:
        // ...
        virtual ~X() { delete[] a; }   
};
```

Now there is no more leak. 

We don't need (and maybe must not?) use `override` for this. The compiler is smart enough and from the doc, the derived destructor always overrides it.

__Always__ make the destructor virtual in classes that are meant to be superclasses:
- Even if the destructor does nothing.
- You never know what the subclass's destructor might do, so you need to make sure the subclass's destructor gets called.
- Also always give your virtual destructor an implementation, even though it might be empty. It will get called by the subclass destructor.

If a class is not meant to be a superclass, then no need to incur the cost of virtual methods needlessly.
- Leave the destructor non-virtual.

```C++
class X final { // Cannot be subclassed
    ...
};
```

Like `override`, `final` is another contextual keyword (right before the brace).

Also from the doc ([cppreference](https://en.cppreference.com/w/cpp/language/virtual#:~:text=result%20to%20B*%20%7D-,Virtual%20destructor,type%20through%20pointers%20to%20base.)):
> Moreover, if the destructor of the base class is not virtual, deleting a derived class object through a pointer to the base class is undefined behavior regardless of whether there are resources that would be leaked if the derived destructor is not invoked, unless the selected deallocation function is a destroying operator delete (since C++20).

> A useful guideline is that the destructor of any base class must be public and virtual or protected and non-virtual, whenever delete expressions are involved, e.g. when implicitly used in `std::unique_ptr` (since C++11).


---
[Heterogeneous Data << ](./problem_21.md) | [**Home**](../README.md) | [>> If a Class Has No Objects](./problem_23.md)
