[Now You've Gone Too Far! << ](./problem_10.md) | [**Home**](../README.md) | [>> Where Do I Even Start](./problem_12.md)

# Problem 11: I Want a Vector of Chars
## **2025-09-25**

Start over? No

Introduce a major abstraction mechanism, **templates**
- Generalize over types

#### Vector.cc

```C++
export module Vector;

namespace CS246E {
    export template<typename T> class Vector {
        size_t n, cap;
        T* theVector;

        public:
            Vector();
            // ...
            void push_back(T n);
            T &operator[](size_t i);
            const T &operator[](size_t i) const;

            using iterator = T*;
            using const_iterator = const T*;
            // etc.
    };

    template<typename T> Vector<T>::Vector() n{0}, cap{1}, theVector{new T[cap]} {}
    template<typename T> void Vector<T>::push_back(T n) {...}
    // etc.
}
```

**Note:** Must put implementation in the interface file.

#### main.cc

```C++
int main() {
    CS246E::Vector<int> v;  // Vector of ints
    v.push_back(1);
    ...
    CS246E::Vector<char> w; // Vector of chars
    w.push_back('a');
    ...
}
```

- **Semantics:**
    - The first time the compile encounters `Vector<int>`, it creates a version of the vector code where `int` replaces `T` and compiles that new class. Similarly for `Vector<char>`.
    - Can't do that unless it has all the details about the class.
    - So implementation must be available , i.e. in the interface file.
    - Can also write bodies inline.

---
[Now You've Gone Too Far! <<](./problem_10.md) | [**Home**](../README.md) | [>> Where Do I Even Start](./problem_12.md)
