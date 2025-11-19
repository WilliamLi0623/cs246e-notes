[A big unit on Object Oriented Design <<](./object_oriented_design.md) | [**Home**](../README.md) | [>> Abstraction over Iterators](./problem_25.md)

# Problem 26: In Which We Challenge Problem 26
## **2025-11-11**

Is `dynamic_cast` really bad style?

Where have we used it?
- `whatIsIt` - breaks if we add more book type.
- `virtual operator=` - doesn't.
  - Only need to compare with your own type, not all types in the hierarchy.
- So `operator=`, at least, was OK.

How might we fix `whatIsIt`? Could have written a virtual method:

```C++
class Book {
    // ...
    virtual void identify() { cout << "Book"; }
};

void whatIsIt(Book *b) {
    if (b) b->identify();
    else cout << "Nothing";
}
```

Works by creating an interface function that is uniform across all `Book` types.
- But what if the interface isn't uniform across all types in the hierarchy?

Inheritance and virtual methods works well when:
- There is an unlimited number of potential specializations of a base abstraction.
- Each following the same interface.

But what if you have the opposite case:
- Small number of specializations, all know in advanced, and unlikely to change.
- Different specializations might have different interfaces.

In the first case - adding a subclass is zero effort.
In the second case - adding a subclass means there is work, but we're not expected to do that, or if we do, we expect the work.

Ex.

```C++
class Turtle: public Enemy {
    void stealShell();
};

class Bullet: public Enemy {
    void deflect();
};
```

Inheritances not uniform. New `Enemy` => new interface, unavoidable work.

So:
- Maybe it's best to regard the set of `Enemy` types as fixed.
- Then maybe dynamic casting isn't so bad.

BUT
- Then maybe inheritance is the wrong mechanism.

If you know that `Enemy` will only be `Turtle` or `Bullet`, and you accept that adding new `Enemy` types will require widespread changes anyway, then consider:

```C++
import <variant>; // Type-safe union
using Enemy = std::variant<Turtle, Bullet>;
// An Enemy is a Turtle or a Bullet. Period.
Enemy e {Turtle{}};
```

Discriminating the value:

```C++
if (holds_alternative<Turtle>(e)) {
    cout << "Turtle";
} else {
    //...
}
```

Extracting the value:

```C++
try {
    Turtle t = get<Turtle>(e);
    // Use t...
} catch (bad_variant_access &) {
    // It's not a Turtle...
}
```

Store as one type and fetch as another => throws.

Variant left uninitialized
- First option default-constructed.
- Compile error if not default constructible.

Options:
1. Pick a type with a default constructor for the first option.
2. Don't leave your variant uninitialized.
3. Use a "dummy" type as the first option. `std::monostate` exists for this purpose.

- Can use to make an "optional" type.

Ex.

```C++
variant<monostate, T> // T or nothing
```

- But also, `std::optional<T>`.

Implementation - we will see later.

Is C++ hard? (No, if you're a client programmer)

But explicit memory management...
- If you need an array, use `vector`
- If you need a heap object, use `unique_ptr`
- Use stack-allocated objects as much as possible

If you follow these you should never have to say `delete` or `delete[]`

But `unique_ptr` doesn't respect IS-A:

```C++
unique_ptr<Base> p = new Derived{}; // OK
p->virtfn(); // Runs derived version, OK
```

BUT

```C++
unique_ptr<Derived> q = ...
unique_ptr<Base> p = std::move(q);  // Wrong
```

Type error, no conversion between `unique_ptr<Derived>` and `unique_ptr<Base>`

Easy to fix:

```C++
template<typename T> 
class unique_ptr {
private:
    T *p;
    // ...
public:
    template<typename U> unique_ptr(unique_ptr<U> &&q): p{q.p} {
            q.p = nullptr;
    }

    template<typename U> unique_ptr &operator=(unique_ptr<U> &&q) {
        std::swap(p, q.p);  // T and U are mutually assignable
        return *this;
    }
};
```

Works for any `unique_ptr` whose pointer is assignment compatible with `this->p`
- Eg. Subtypes of `T`, but not super-types of `T`
- Eg. `char*` to `string`

"But I want two smart pointers pointing at the same object!"
- Why? The pointers that *owns* the object should be a `unique_ptr`
    - All others should be raw pointers

When would you want true shared ownership?

Recall in Racket:

```racket
(define l1 (cons 1 (cons 2 (cons 3 empty))))
(define l2 (cons 4 (rest l1)))
```

```
   +---+---+    +---+---+   +---+---+
l1 | 1 | ------>| 2 | ----->| 3 | \ |
   +---+---+    +---+---+   +---+---+
                  ^
   +---+---+      |
l2 | 1 | ---------+
   +---+---+
```

Shared data structures are a nightmare in C. How can we ensure each node is freed exactly once?

Easy in garbage collected languages.

What can C++ do?

```C++
template <typename T> class shared_ptr {
    T *p;
    int *refcount;
public:
    // ...
};
```

- `refcount` counts how many `shared_ptr`s point at `*p`.
- Updated each time a `shared_ptr` is initialized/assigned/destroyed
- `refcount` is shared among all shared pointers that point to `*p`
- `p` is only deleted if its `refcount` reaches `0`
- implementation details - left to you

```C++ 
struct Node {
    int data;
    shared_ptr<Node> next;
};
```
However, only use it as you *desperately* need it, because it has a larger overhead than `unique_ptr` and ownership is messed up.

Now deallocation is as easy as garbage collection

Also watch: cycles

```
   +---+---+    +---+---+  
   | 1 | ------>| 2 | -----+
   +---+---+    +---+---+  |
     ^                     |
     |                     |
     +---------------------+
```

If you have cyclic data, you may have to break the cycle (or use `weak_ptrs`)

Also watch:

```C++
Book *p = new ...
shared_ptr<Book> p1 {p};
shared_ptr<Book> p2 {p}; // Will compile but UB
```
- `shared_ptr`s are not mind-readers
- `p1` and `p2` will not share a `refcount` (BAD)
- If you want 2 `shared_ptr`s at an object, create one `share_ptr` and copy it
- From the [doc](https://en.cppreference.com/w/cpp/memory/shared_ptr):
  > The ownership of an object can only be shared with another shared_ptr by copy constructing or copy assigning its value to another shared_ptr. Constructing a new shared_ptr using the raw underlying pointer owned by another shared_ptr leads to undefined behavior.

BUT

You can't `dynamic_cast` these pointers, but that doesn't stop us from building one 😉

First, give `shared_ptr` a new ctor:
```C++
template <typename T> class shared_ptr {
    // ...
public:
    // creates a shared_ptr that shares the refcount with x, but points to raw
    template <typename U> shared_ptr(shared_ptr<U> &x, T* raw); // exercise
};
```

```C++
template<typename T, typename U> 
shared_ptr<T> dynamic_pointer_cast(const shared_ptr<U> &r) {
    if (auto p = dynamic_cast<T*>(r.get())) // yes, 1 equal because we want to test if cast succeeds
        return shared_ptr<T>(r, p);
    return shared_ptr<T>{ nullptr };
}
```

Similarly `const_pointer_cast`, `static_pointer_cast`

---
[A big unit on Object Oriented Design <<](./object_oriented_design.md) | [**Home**](../README.md) | [>> Abstraction over Iterators](./problem_25.md)
