[I Want a Vector of Chars <<](./problem_11.md) | [**Home**](../README.md) | [>> I Want a Vector of Posns](./problem_13.md)

# Problem 12: Where Do I Even Start
## **2025-09-25**

Long sequence of push_backs is clunky.

```C++
a[] = {1, 2, 3, 4, 5};  // Array    :)

CS246E::Vector<int> v;  // Vector   :(
v.push_back(1);
// ...
```
    
Goal: better initialization

```C++
template<typename T> class Vector {
    // ...
    public:
        Vector(): // ...
        Vector(size_t n, T x = T{}): n {n}, cap {n == 0 ? 1 : n}, theVector{new T[cap]} {
            for (size_t i = 0; i < n; ++i) {
                theVector[i] = x;
            }
        }
};
```

Now:

```C++
Vector<int> v;  // Empty
Vector<int> w{5};   // 0, 0, 0, 0, 0
Vector<int> z{3, 5};    // 5, 5, 5
```

**Notes:** `T{}` (default constructor) means `0` if `T` is a built-in type.

Better - what about true array-style initialization?

```C++
import <initializer_list>;
template <typename T> class Vector {
    // ...
    public:
        Vector() ...;
        Vector(size_t n, T i = T{}) ...
        Vector(std::initializer_list<T> init): 
            n{init.size()}, cap{init.size()}, theVector{new T[cap]} {
                size_t i = 0;
                for (auto t: init) theVector[i++] = t;
            }
};
```

Gives us:

```C++
Vector<int> v {1, 2, 3, 4, 5};  // 1, 2, 3, 4, 5
Vector v {1,2,3,4,5};   // Starting from C++17, can omit type if it can be inferred
Vector<int> v;  // Empty, int is required here
Vector<int> v {5};   // 5
Vector<int> v {3, 5};    // 3, 5
```

## **2025-09-30**

Default constructors take priority over initializer lists, which take priority over other constructors

To get the other constructor to run: **round bracket initialization**

```C++
Vector<int> v(5);   // 0, 0, 0, 0, 0
Vector<int> v(3, 5);    // 5, 5, 5
```

A note on cost: item in an initializer list are stored in contiguous memory (begin method returns a pointer).
- So we are using one array to build another (2 copies in memory).

Also note:
- Initializer lists are meant to be immutable.
- Do not try to modify their contents.
- Do not use `initializer_list`s as standalone data structures.
- Only one allocation in vector, not several.
- No doubling + reallocating as there was with a sequence of `push_back`s.
- If general, if you know how big your vector will be, you can save reallocation cost by requesting the memory up front.

```C++
template<typename T> class Vector {
    ...
    public:
        ...
        void reserve(size_t newCap) {
            if (cap < newCap) {
                T *newVec = new T[newCap++];
                for (size_t i = 0; i < n; ++i) newVec[i] = theVector[i];
                delete[] theVector;
                theVector = newVec;
                cap = newCap;
            }
        }
};
```

**Exercise:** rewrite `Vector` such that `push_back` uses `reserve` instead of `increaseCap`

```C++
Vector<int> v;
v.reserve(10);
v.push_back(__);    // Can do 10 push_backs without needing to reallocate
// ...
```

---
[I want a vector of chars <<](./problem_11.md) | [**Home**](../README.md) | [>> Actually ... I Want a Vector of Posns](./problem_13.md)
