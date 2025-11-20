[Generalize the Visitor pattern <<](./problem_26.md) | [**Home**](../README.md) | [>> I want to print the unprintable!](./problem_28.md)

# Problem 30: I want an even faster vector
## **2025-11-19**

In the good old days of C, you could copy an array (even an array of structs!) very quickly by calling a function `memcpy` (similar to `strcpy`, but for arbitrary memory, not just characters).

`memcpy` was probably written in assembly, and was as fast as the machine could possibly be.

Now in C++, copies invoke copy constructors, which are costly function calls.

Can we go back to these simpler times?

In C++, a type is considered **POD (plain old data)** if it:
- Has a trivial default constructor (equiv. to `= default`).
- Is trivially copyable.
  - Big 5 all have default implementations.
- Is standard layout.
    - No virtual methods or bases.
    - All members have the same visibility.
    - No reference members.
    - No fields in both class & subclass, or in more than one base class.

For POD types, semantics is compatible with C, and `memcpy` is safe to use.

How can we use it? - Only safe to use if `T` is a POD type

## **2025-11-20**

_One option:_

```C++
template<typename T> class vector {
private:
    size_t n, cap;
    T *theVector;
public:
    // ...
    vector(const vector &other): 
        theVector{static_cast<T*>(operator new (other.n * sizeof(T)))}, n{other.n}, cap{other.cap} {
        // value is true if T is pod, false otherwise
        if (std::is_pod_v<T>) {
            memcpy(theVector, other.theVector, n * sizeof(T));
        } else {
            // as before
        }
    }
}
```

Works... But condition is evaluated at run-time, but the result is known at compile-time (compiler may or may not optimize)

_Second option (at compile-time, outdated):_
- Make two versions of the constructor, both templates, have exactly one of them be valid based on `std::is_pod_v<T>`. Then SFINAE will pick up the valid one.

```C++
template<typename T> class vector {
// ...
public:
    template<typename X = T>
    vector(enable_if<std::is_pod_v<X>, const T&>::type other): 
        theVector{static_cast<T*>(operator new (other.n * sizeof(T)))}, n{other.n}, cap{other.cap} {
            memcpy(theVector, other.theVector, n * sizeof(T));
        }

    template<typename X = T>
    vector(enable_if<!std::is_pod_v<X>, const T&>::type other):
        theVector{static_cast<T*>(operator new (other.n * sizeof(T)))}, n{other.n}, cap{other.cap} {
            // original implementation
        }
}
```

How does it work?

```C++
template<bool b, typename T> struct enable_if;
template<typename T> struct enable_if<true, T> {
    using type = T;
};
```

_Third option (concepts):_

```C++
template <typename T> class vector {
    // ...
public:
    template<typename X = T> requires std::is_pod_v<X>
    vector(const T &other): 
        theVector{static_cast<T*>(operator new (other.n * sizeof(T)))}, n{other.n}, cap{other.cap} {
            memcpy(theVector, other.theVector, n * sizeof(T));
        }

    template<typename X = T> requires !std::is_pod_v<X>
    vector(const T &other):
        theVector{static_cast<T*>(operator new (other.n * sizeof(T)))}, n{other.n}, cap{other.cap} {
            // original implementation
        }
};
```

So this compiles, but it crashes. Why? If you put debugging print statements, in these copy constructors, they don't print.
- We're getting the provided copy constructor => shallow copies.
- These templates are not enough to suppress the generation of the provided copy constructor. - And a non-templated match is always preferred over a templated match.

What can we do about this? Could try:

```C++
vector (const vector &) = delete;
```

Not allowed, can't disable the copy constructor and still write one.

Solution that actually works: **overloading**

```C++
template<typename T> class vector {
    // ...
    // A dummy structure just give me a type so I can overload on.
    struct dummy{};
public:
    vector(const vector &other): vector{other, dummy{}} {}

    template<typename X = T> requires is_pod_v<X>
    vector(const vector &other, dummy) { ... }

    template<typename X = T> requires !is_pod_v<X>
    vector(const vector &other, dummy) { ... }
};
```

_Forth option (constexpr if):_
If you just want to use a compile-time value to choose between implementations, there is an easier way.

```C++
template <typename T> class vector {
    // ...
    struct dummy{};
public:
    vector (const vector &other): theVector{static_cast<T*>(operator new (other.n * sizeof(T)))}, n{other.n}, cap{other.cap} {
        if constexpr (std::is_pod_v<T>) {
            memcpy(theVector, other.theVector, n * sizeof(T));
        } else {
            // original implementation
        }
    }
};
```

For constexpr if:
- Condition must be a compile-time value.
- Non-matching branch is **discarded**.

## Move / Forward implementation

<u>Aside:</u> We now have enough machinery to implement `std::move` and `std::forward`.
- _`std::move`_ - treat a lvalue as a rvalue (cast).
- First attempt
```C++
template<typename T> T &&move(T & x) {
    return static_cast<T &&>(x);
}
```

Doesn't quite work, `T&&` is a universal reference, not necessarily an rvalue reference. If `x` was a lvalue reference, `T&&` is a lvalue reference.
- Need to make sure `T` is not an lvalue reference
    - If `T` is an lvalue reference, get rid of the reference
    - Basically we get rid of all the ref to get the bare type with `std::remove_reference`.

```C++
template<typename T> inline typename std::remove_reference<T>::type&& move(T&& x) {
    return static_cast<typename std::remove_reference<T>::type &&>(x);
    // turns T&, T&& into T
}
```

**Exercise:** write `remove_reference`

**Q:** can we save typing and use `auto`? Ex.

```C++
template<typename T> auto move(T &&x) { ... }
```

**A:** No! By-value auto throws away reference and outer const
- Recall that `auto x = y` gives x the same type as the **value** of y (i.e, the type of `y` as if it was copied)

```C++
int z;
int &y = z;
auto x = y; // x is an int

const int &w = z;
auto v = w; // int
```

By reference, `auto &&` is a universal reference, so the code from the question section works (it does compile), but not as we expected.

But still... is there a way to do type deduction without losing ref and const?

Need a type definition rule that doesn't discard references.

The answer is yes, by using `decltype`. It produces the type its argument was declared to have.

**Note**: The argument of `decltype` is never evaluated, only type-checked.

```C++
decltype(var)  // returns the declared type of the variable
decltype(expr) // returns lvalue or rvalue, depending on whether the expr is an lvalue or rvalue

int z;
int &y = z;
decltype(y) x = z;  // x is an int&, auto would only give you int
x = 4;  // Affects z

/* Path/Example 1 */
auto z;
x = 4;  // Does not affect z

/* Path/Example 2 */
decltype(z) s = z;  // s is an int
s = 5; // Does not affect z

/* Path/Example 3 */
decltype((z)) r = z;    // r is an int&, since (z) is a ref, since () is an expression
r = t;  // Does affect z

decltype(auto) - perform type deduction, like auto, but use the decltype rules
```

Can we do type deduction using the `decltype` rules instead of the auto rules? 

Yes - use `decltype(auto)`

```C++
template<typename T> decltype(auto) move(T &&x) {
    return static_cast<std::remove_reference_t<T>&&>(x);
}
```

- Let's try implementing `std::forward`
_`std::forward`_
```C++
template<typename T> T&& forward(T &&x) {
    return static_cast<T&&>(x);
}
```
- Doesn't seem right - casting `x` to its own type.

**Reasoning:**
- If `x` is an lvalue, `T&&` is an lvalue reference
- If `x` is an rvalue, `T&&` is an rvalue reference

Doesn't work, `forward` is called on expressions that are lvalues, that may point at rvalues.

```C++
template<typename U> void f(U&& y) {
    ... forward(y) ...  // y is an lvalue
}
```

`forward(y)` is will _always_ yield an lvalue reference.

In order to work, `forward` must know what type (including l/rvalue) was deduced for `y`, ie. needs to know `U`.

So in principle, `forward<U>(y)` would work.

**2 Problems:**
- Supplying `T` means `T&&` is no longer universal
- Want to prevent the user from omitting `<T>`

**Instead:** separate lvalue/rvalue cases

```C++
template<typename T> 
inline constexpr T&& forward(std::removed_reference_t<T>&x) noexcept {
    return static_cast<T&&>(x);
}

template<typename T> 
inline constexpr T&& forward(std::removed_reference_t<T>&&x) noexcept {
    return static_cast<T&&>(x);
}
```

---
[Generalize the Visitor pattern <<](./problem_26.md) | [**Home**](../README.md) | [>> I want to print the unprintable!](./problem_28.md)