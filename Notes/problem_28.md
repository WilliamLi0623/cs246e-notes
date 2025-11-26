[Shared Ownership <<](./problem_27.md) | [**Home**](../README.md) | [>> Do You Expect Me to be Able to Do Something Like That?](./problem_29.md)

# Problem 28: Abstraction Over Iterators
## **2025-11-13**

I want to jump ahead `n` spots in my container.

```C++
template<typename Iter> Iter advance(Iter it, size_t n);
```

How should we do it?

```C++
for (size_t i = 0 ; i < n ; ++ i) ++ it;
return it;
```

But this is so damn slow (`O(n)` time), can't we say `it += n`?

Depends:
- For `vector`, yes (O(1) time).
- For `list`, no (`+=` not supported, and if it was it'd still be in a loop).

Related - Can we go backwards (`--`, `-=`)?
- `vector`, yes.
- `list` no.
- `std::list` yes.

So all iterators support `!=`, `*`, `++`, but some iterators support other operations.

- `list::iterator` - called a **forward iterator**, can only go one step forward.
- `std::list::iterator` - called a **bidirectional iterator**.
- `vector::iterator` - called a **random access iterator** can go anywhere (arbitrary pointer arithmetic).

## **2025-11-18**

How can we write `advance` to use `+=` for random access iterators and a loop for forward iterators.

Since we have different kinds of iterators let's create a type hierarchy:

```C++
struct forward_iterator_tag: input_iterator_tag{};
struct bidirectional_iterator_tag: forward_iterator_tag{};
struct random_access_iterator_tag: bidirectional_iterator_tag{};
```

These are called `tag`, a class or struct with no fields and an empty body. This technique is called **tag dispatching**, done until C++17, and replaced by **concepts** in C++20 (ref-link [here](https://internalpointers.com/post/writing-custom-iterators-modern-cpp))

To associate each iterator class with a `tag`, could use inheritance:

Ex.

```C++
class list {
    // ...
    public:
        class iterator: public forward_iterator_tag {
            // ...
        };
};
```

But: this makes it hard to ask what kind of iterator we have (can't `dynamic_cast`, no `vtables`).
- Doesn't work for iterators that aren't classes (eg. pointers).

Instead make the `tag` a member:

```C++
class list {
    // ...
    public:
        class iterator { 
            // ...
            public:
                using iterator_category = forward_iterator_tag;
                // typedef forward_iterator_tag iterator_category;
        };
};
```

**Convention:** every iterator class will define a type member called `iterator_category`.
- Still doesn't work for iterators that aren't classes.
- But we aren't done yet.

Make a template that associates every iterator type with its category.

```C++
template<typename It> struct iterator_traits {
    using iterator_category = It::iterator_category;
    // typedef typename It::iterator_category if older C++
};

// Ex.
iterator_traits<List<T>::iterator>::iterator_category // forward_iterator_tag
```

And then we can provide a specialized version for **pointers**:

```C++
template<typename T> struct iterator_traits<T*> {
    using iterator_category = random_access_iterator_tag; // put the tag name directly here rather than through It::iterator_category, apparently because we don't have a class for pointers
};
```

For any iterator type `T`, `iterator_traits<T>::iterator_category` resolves to the `tag` struct for `T` (including if `T` is a pointer).

What do we do with this?

Want:

```C++
template <typename Iter>
Iter advance(Iter it, int n) {
    if (typeid(typename iterator_traits<Iter>::iterator_category) 
        == typeid(random_access_iterator_tag)) {
        return it += n;
    } else {
        // ...
    }

}
```

- But this won't compile.
- If the iterator is not random access, and doesn't have a `+=` operator, `it += n` will cause a compilation error, even though it will never be used.
- Moreover, the choice of which implementation to use is being made at run-time, when the right choice is known at compile-time.

To make a compile-time decision - overloading.
- Make a dummy parameter with type of the iterator `tag`.

```C++
template <typename Iter>
Iter doAdvance(Iter it, int n, random_access_iterator_tag) {
    return it += n;
}

template <typename Iter>
Iter doAdvance(Iter it, int n, bidirectional_iterator_tag) {
    if (n > 0) for (int i = 0 ; i < n ; ++ i) ++ it;
    else if (n < 0) for (int i = 0 ; i > n ; -- i) -- it;
    return it;
}

template <typename Iter>
Iter doAdvance(Iter it, int n, forward_iterator_tag) {
    if (n >= 0) {
        for (int i = 0; i < n; ++ i) ++ it;
        return it;
    }
    throw SomeError{};
}
```

Finally, create a wrapper function to select the right overload:

```C++
template <typename Iter>
Iter advance(Iter it, int n) {
    return doAdvance(it, n, typename iterator_traits<Iter>::iterator_category{});
}
```

Now the compiler will select the fast `doAdvance` for random access iterators, the slow `doAdvance` for bidirectional iterators, and the throwing `doAdvance` for forward iterators.

These choices made at **compile-time** - no runtime cost.

Using template instantiation to perform compile-time computation - called **template metaprogramming**

C++ templates form a functional language that operates at the level of types.
- Express conditions by overloading, repetition via recursive template instantiation.
- Also turing complete.
  - There are template instantiation that don't terminate, because expressing arbitrary computations implies expressing non-terminating computations.
  - As a result, compiler usually put a limit on recursion depth.

Example:

```C++
template <int N> struct Fact {
    static const int value = N * Fact<N - 1>::value;
};

template<> struct Fact<0> {
    static const int value = 1;
};

int x = Fact<5>::value; // 120 - evaluated at compile-time!!!
```

But for compile-time computation of values, C++ offers a more straightforward facility:

```C++
constexpr int fact(int n) {
    if (n == 0) return 1;
    else return n * fact(n - 1);
}
```

- `constexpr` functions
    - evaluate this at compile-time if `n` is a compile-time constant.
    - else evaluate at runtime.

Can we spam `constexpr` everywhere?
- A `constexpr` function must be something that actually can be evaluated at compile-time.
  - Can't be virtual.
  - Can't mutate non-local variables.
  - Etc.

---
[Shared Ownership <<](./problem_27.md) | [**Home**](../README.md) | [>> Do You Expect Me to be Able to Do Something Like That?](./problem_29.md)