[Tampering << ](./problem_8.md) | [**Home**](../README.md) | [>> Now You've Gone Too Far!](./problem_10.md) 

# Problem 9: Walk Faster
## **2025-09-23**

Consider the two implementations Vector and List:

```C++
vector v;
v.push_back(...);
// ...

for (size_t i = 0; i < v.size(); ++i) {
    std::cout << v[i] << std::endl;     // O(1)
}
```

- Array access - efficient
- O(n) traversal

```C++
list l;
l.push_front(...);
// ...

for (size_t i = 0; i < l.size(); ++i) {
    std::cout << l[i] << std::endl;     // O(n)
}
```

- O(n^2) traversal.
- No direct access to "next" pointers, we can't make this fast, how can we do efficient iteration?

**Design Patterns**
- Well known solutions to well-studied problems.
- Adapted to suit needs.

**Iterator Pattern**
- For efficient iteration over a collection, without exposing the underlying structure.

**Idea:** Create a class that "remembers" where you are in the list (abstraction of a pointer)

**Inspiration:** C

```C
for (int *p = arr; p != arr + size; ++p) {
    printf("%d\n", *p);
}
```

```C++
class list {
    struct Node {...};
    Node *theList = nullptr;
    size_t len = 0;

    public:
        class iterator {
            Node *p;

            public:
                iterator(Node *p): p{p} {}
                bool operator!=(const iterator &other) const {return p != other.p;}
                int &operator*() {return p->data;}
                iterator &operator++() {    // Prefix version
                    p = p->next;
                    return *this;
                }
        }

    iterator begin() {return iterator{theList};}
    iterator end() {return iterator{nullptr};}
};
```

```C++
list l;
// ...
for (list::iterator it = l.begin(); it != l.end(); ++it) {
    std::cout << *it << " ";
}
```

**Q:** Should `list::begin` and `list::end` be `const` methods?
**Consider:**

```C++
ostream &operator<<(ostream &out, const list &l) {
    for (list::iterator it = l.begin(); it != l.end(); ++it) {
        ...
    }
    ...
}

``` 

Won't compile if `begin`/`end` are not `const`

```C++
ostream &operator<<(ostream &out, const list&l) {
    for (list::iterator it = l.begin(); it != l.end(); ++it) {
        out << *it << '\n';
        ++*it;  // increment items in the list
    }
}
```

Will compile but shouldn't, the list is supposed to be `const`, but `*` returns as non-`const`.
- In the case that `l` is `const`, deref operator on the iterator should return a `const` reference. 
- But if `l` is **not** `const`, deref operator on the iterator should return a non-`const` ref.

**Conclusion:** Iteration over `const` is different from iteration over non-`const`
- Make a second iterator class.

```C++
class list {
    struct Node {...};
    Node *theList;

    public:
        class iterator {
            Node *p;

            public:
                iterator(Node *p): p{p} {}
                bool operator!=(const iterator &other) const { return p != other.p; }
                int &operator*() const { return p->data; }
                iterator &operator++() {    // Prefix version
                    p = p->next;
                    return *this;
                }
        };

        class const_iterator {
            Node *p;

            public:
                const_iterator(Node *p): p{p} {}
                bool operator!=(const const_iterator &other) const { return p != other.p; }
                const int &operator*() const { return p->data; } // return a const ref
                const_iterator &operator++() {
                    p = p->next;
                    return *this;
                }
        };

        iterator begin() { return iterator{theList}; }
        iterator end() { return iterator{nullptr}; }
        const_iterator begin() const { return const_iterator{theList}; }
        const_iterator end() const { return const_iterator{nullptr}; }
};
```

Exercise: `cbegin` and `cend`.

Works now:

```C++
ostream &operator<<(ostream &out, const list&l) {
    for (list::const_iterator it = l.begin(); it != l.end(); ++it) {
        out << *it << '\n';
        ++*it;  // increment items in the list
    }
}
```
However:

```C++
list::const_iterator it = l.begin();    // this is mouthful
```

Shorter:

```C++
ostream &operator<<(ostream &out, const list &l) {
    for (auto it = l.begin(); it != l.end(); ++it) {
        out << *it << '\n';
    }
    return out;
}
```

For `auto x=expr;`:
- Saves writing down `x`'s type.
- `x` will be given the same type as `expr`'s value.

Even shorter:

```C++
ostream &operator<<(ostream &out, const list &l) {
    for (auto n : l) { 
        out << n << ' ';   
    }
    return out;
}
```

This is a range-based `for` loop
- Available for any class with:
    - Methods (or functions) `begin()` and `end()` that return an iterator object.
    - The iterator class must support unary`*`, prefix`++`, and `!=`.

**Note:**
- `for (auto n: l) ++n;`
    - `n` is declared by value.
    - `++n` increments n, not the list items (does not mutate the list).
- `for (auto &n : l) ++n;`
    -  `n` is a reference, will update list elements (mutate the list).
- `for (const auto &n : l) ____;`
    - `const` reference, cannot be mutated.

One small encapsulation problem:

**Client:** `list::iterator it {nullptr}`
- Forgery, create an end iterator without calling `end();`

```C++
list::iterator it{nullptr};
```

**To fix:** make iterator constructor private

**BUT:** List can't create iterators either  

**Solution: friendship**

```C++
class list {
    // ...
    public:
        class iterator {
            // ...
            Node *p;
            iterator(Node *p): p{p} {}           
            public:
            // ...
            friend class list;  // list has access to all iterator's/const_iterator's implementation
        };

        class const_iterator {  // Same (friend class list)
            // ...
            Node *p;
            const_iterator(Node *p): p{p} {}
            public:
            // ...
            friend class list;  // list has access to all iterator's/const_iterator's implementation
        };
};
```

C++ advice (or life advice): Limit your friendship, they weaken encapsulation (the implementation is exposed to the classes that are friends with this class, and changes will affect those classes).

Recall: Encapsulation + Iterators for linked list

Can do the same for vectors:

```C++
class vector {
    size_t size, cap;
    int *theVector;
    
    public:
        class iterator {
            int *p;
            // ...
        };

        class const_iterator {
            // ...
        };

        iterator begin() {return iterator{theVector};}
        iterator end() {return iterator{theVector + n};}
        const_iterator begin() const {return const_iterator{theVector};}
        const_iterator end() const {return const_iterator{theVector + n};}
};
```

- Notice that, in case of list, the iterator pattern helps us to not allow users to break our structure. However, in case of the vector, it doesn't really help anything because it's just an array, user can't break it at all.
- Could do this, OR:

```C++
typedef int *iterator;
typedef const int *const_iterator;

---

// or if you hate typedef syntax
using iterator = int*;
using const_iterator = const int*;

// i.e, use ordinary pointers as iterators
iterator begin() {return theVector;}
iterator end() {return theVector + n;}
```

---
[Tampering << ](./problem_8.md) | [**Home**](../README.md) | [>> Now You've Gone Too Far!](./problem_10.md)