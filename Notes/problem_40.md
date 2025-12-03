[Total Control <<](./problem_35.md) | [**Home**](../README.md) | [>> A fixed-size object allocator](./problem_37.md)

# Problem 39: Total Control over Vectors and Lists

## **2025-11-27**

How can we incorporate custom allocation into our containers.

**Issue:** May want different allocators for different kinds of vectors.

**Solution:** Make the allocator an argument to the template.

Since most users won't write allocators, we'll need a default value.

**Template Signature:**

```C++
template<typename T, typename Alloc = std::allocator<T>> class vector { ... };
```

`std::allocator<T>` provides default allocator functionality for objects of type `T`.

```C++
template<typename T> struct allocator {
    // no data members, stateless, has no fields
    using value_type = T;

    // trivial - no data members
    allocator() noexcept {}

    // trivial - no data members
    allocator(const allocator &) noexcept {}

    // assigning allocator of another type
    template<typename U> allocator(const allocator<U> &) noexcept {}

    // trivial
    ~allocator() {}

    // making life easier
    using pointer = T*;
    using reference = T&;
    using const_pointer = const T*;
    using const_reference = const T&;
    using size_type = size_t;
    using difference_type = ptrdiff_t;

    pointer address(reference x) const noexcept { return &x; }
    const_pointer address(reference x) const noexcept { return &x; }

    pointer allocate(size_type n) { return ::operator new(n * sizeof(T)); }
    void deallocate(pointer p, size_type n) { ::operator delete(p); }

    template<typename U, typename ...Args>
    void construct(U *p, Args &&... args) {
        // placement new here
        ::new(static_cast<void*>(p)) U(std::forward<Args>(args)...);
    }

    template<typename U> void destroy(U *p) { p->~U(); }

    size_type max_size() const noexcept {
        return std::numeric_traits<size_type>::max(sizeof(value_type));
    }
};
```

If you want to write an allocator for an STL container, this is its interface (+ more stuff)

**Note:** `operator new` takes a number of _bytes_, but `allocate` takes a number of _objects_.

What happens if a vector is copied? Copy the allocator? What happens if you copy an allocator? Can 2 copies allocate/deallocate each other's memory?

- C++03 allocators must be stateless - copying allocators is allowed and trivial
- C++11 allocators can have state, can specify copying behaviour via allocator traits

Adapting `vector`:

- `vector` has a field `Alloc alloc`;
- everywhere `vector` calls
  - `operator new`, replace with `alloc.allocate`
  - `placement new`, replace with `alloc.construct`
  - `dtor` explicitly, replace with `alloc.destroy`
  - `operator delete`, replace with `alloc.deallocate`
  - takes an address, replace with `alloc.address`
- Details: exercise

Can we do the same with list?

```C++
template<typename T, typename Alloc = std::allocator<T>> class list { ... }
```

Correct so far... but curiously, `Alloc` will never be used to allocate memory in a list.

Why not? - `list`s are node-based, which means you don't want to actually allocate `T` objects; you want to allocate nodes (which contains `T` objects and pointers).

- But `Alloc` allocates `T` objects.

How do we get an allocator for nodes?

We could give `MyAlloc` a member that produces a different specialization of `MyAlloc`:

```C++
template <typename T> class MyAlloc {
public:
    // ...
    template <typename U> struct rebind {
        using other = MyAlloc<U>;
    };
}
```

Within `list` - to create an allocator for nodes as a field of list:

```C++
typename MyAlloc::rebind<Node>::other alloc; // Use this allocator
```

Then it's the same as `vector`.
- Could have been done with a template template parameter.

---

[Total Control <<](./problem_35.md) | [**Home**](../README.md) | [>> A fixed-size object allocator](./problem_37.md)
