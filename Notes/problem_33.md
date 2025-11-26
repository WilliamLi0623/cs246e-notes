[Variants Revisited <<](./problem_32.md) | [**Home**](../README.md) | [>> Variants Revisited](./problem_34.md)
# Problem 33 - Collecting Stats
## **2025-11-25**

I want to know how many `Student`s I created.

```C++
class Student {
    int assns, mt, final;
    inline static int count = 0; // static - associated with the class - not one per object
public:
    Student (...) { ++count; }
    static int getCount() { return count; } // static member functions - they have no `this`, can only access static members or call static member functions
};
```

Before `C++17`, need to define static fields outside the class:

```C++
int Student::count = 0;
```

Now with `inline static`, can define and initialize inside the class.

```C++
Student s1{...}, s2{...}, s3{...};

std::cout << Student::getCount(); // prints 3
```

Now, I want to count objects in other classes. How can we abstract this solution into a reusable code?

```C++
template<typename T> struct Count {
    inline static int count = 0;
    Count() { ++count; }
    Count (const Count &) { ++count; }
    Count (Count &&) { ++count; }
    ~Count() { --count; }
    static int getCount() { return count; }
};
```

```C++
class Student : Count<Student> {
    int assns, mt, final;
public:
    Student (...): ... {}
    using Count::getCount; // bring getCount into Student's scope
};
```

**Private Inheritance**
  - Inherits `Count`'s implementation without creating an "is-a" relationship.
  - Members of  `Count` become private in `Student`.

Now we can easily add it to other classes:

```C++
class Book : Count<Book> {
    // ...
    public:
        using Count<Book>::getCount;
};
```

Why is `Count` a template?
- So that for each class `C`, `class C : Count<C>` creates a new, unique instantiation of `Count`, for each `C`.

Gives class `C`  its own `count`, vs. sharing one counter for all subclasses.

This technique - Inheriting from a tamplate specialized by yourself.
- May look weird, but happens enough to have its own name: **Curiously Recurring Template Pattern (CRTP)**.
  
What is `CRTP` good for?

---
[Variants Revisited <<](./problem_32.md) | [**Home**](../README.md) | [>> Resolving Method Overrides at Compile-Time](./problem_34.md)