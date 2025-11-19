[Abstraction over Iterators <<](./problem_25.md) | [**Home**](../README.md) | [>> I want an even faster vector](./problem_27.md)

# Problem 29: Do you expect me to be able to do something like that?

## **2025-11-18**

Template tricks were kind of an accident. Templates weren't originally designed for compile-time tricks.

But now that it's an accepted thing, can it be made easier?

Let's start by advancing random access iterators.
- If the iterator is random acccess, use `+=`.
- Else the code should not compile.

```C++
template<typename Iter> requires std::is_same_v<iterator_traits<Iter>::iterator_category, random_access_iterator_tag>
Iter advance(Iter it, size_t n) {
    it += n;
    return it;
}

template<typename T, typename U> struct is_same {
    static const bool value = false;
};

template<typename T> struct is_same <T,T> {
    static const bool value = true;
};

template<typename T, typename U> constexpr bool is_same_v = is_same<T,U>::value;
```

- `requires` clause constraints `Iter` such that the stated condition must be true.

What happens if we try to advance a non-random access iterator? `advance (list.begin(),4)`.

If we didn't have the `requires` clause:
- `No operator += for list::iterator`.
  - (Typically long output).
- Now that we do - compiler says `list::iterator` fails the `requires` clause.

## **2025-11-19**

We like abstraction - Can we name this constraint?
- Called a **concept**.

```C++
template<typename Iter>
concept RandomAccessIterator = std::is_same_v<iterator_traits<Iter>::iterator_category, random_access_iterator_tag>;
```

We can now write:

```C++
template<typename Iter> requires RandomAccessIterator<Iter>
Iter advance(Iter it, int n) {
    return it+=n;
}
```

OR:

```C++
template<RandomAccessIterator Iter>
Iter advance(Iter it, int n) {
    return it += n;
}
```

Can we write

```C++
RandomAccessIterator advance(RandomAccessIterator it, int n) {
    return it += n;
}
```

No - `RandomAccessIterator` is not a type, it's a constraint on a type.

And code making use of conecepts is a template.
- This advance looks like an ordinary function.

We can do this:

```C++
RandomAccessIterator auto advance(RandomAccessIterator auto it, int n) {
    return it += n;
}
```

Can write concepts for the other categories.

```C++
BidirectionalIterator auto advance(BidirectionalIterator auto it, int n) {
    // ...
}

ForwardIterator auto advance(ForwardIterator auto it, int n) {
    // ...
}
```

Now if, for example, an iterator's category is `BidirectionalIterator`, the `BidirectionalIterator` version will compile and the other two will fail the template instantiation.

So is this program well-formed? Yes!

C++ rule: **SFINAE**
- Substitution Failure Is Not An Error.

In other words - if `t` is a type and `template<typename T> ... f (...) {...}` is a template function, and substituting `t` for `T` results in an invalid function, the compiler does **not** signal an error - it just removes that instantiation from consideration during overload resolution.

On the other hand - if **no** version of the function is in scope to handle the overload call, that is an error.
- Only applies to template functions.

What if we tell lies? What is we give the class the `RandomAccessIterator` category, but don't give a `+=` operator?

The function will pass concept check, but compilation will fail at the attempted use of `+=`.
- The "old" error checking, prior to concepts.

We could expand our definition of `RandomAccessIterator` to include **both** the right category **and** the needed operations.

```C++
template<typename T> concept RandomAccessIterator = 
    std::is_same_v<iterator_traits<T>::iterator_category, random_access_iterator_tag> &&
    requires (T it, T other, int n) {
        {it != other} -> std::same_as<bool>;
        {++it} -> std::same_as<T>;
        {it += n} -> std::same_as<T>;
    };
```

Etc. for other iterator types.

Now a type with the right category but is missing the needed operations will fail the concept check.

Q: Do we need the iterator tag any more then? Are the listed operations enough?
A: The tag is still valuable. Provides semantic information about the operations. Suppose an iterator is `BidirectionalIterator` but also has `+=` for some unrelated purpose. Need the tag to keep it from being treated as `RandomAccessIterator`.

# Problem 26: Generalize the Visitor Pattern
## **2021-11-18**

Recall the visitor pattern:
```C++
class Book {
    // ...
public:
    virtual void accept (BookVisitor &v) { v.visit(*this); }
};

class Test: public Book {
    // ...
};

class BookVisitor {
public: 
    virtual void visit (Book& b) = 0;
    virtual void visit (Text& t) = 0;
    virtual void visit (Comic& c) = 0;
}
```
- Can we automate this process of creating `visit`? I mean we have **template metaprogramming**

Consider the following:

_visitor.h_
```C++
// right now it would work no matter how many arguments I supply, we will specialize the template for the case when the list of arguments is not empty, so this will only trigger when this is empty
template <typename...> class Visitor {
public:
    void visit();
    virtual ~Visitor() {}
};

// this template has at least 1 known first item
template <typename T, typename... Ts> class Visitor<T, Ts...> : public Visitor<Ts...> {
public:
    // here I inherit whatever visit my parent has, bring parents' visit to the scope
    using Visitor<Ts...>::visit;
    // and I add a visit method that takes a T&
    // for the code above, if we unwind the recursion, we know at each level we would get a visit, until
    // we reach the top level (which is the previous Visitor class), which we would only
    // have a visit method that takes in nothing (trivial case)
    virtual void visit(T& b) = 0;
}
```
- You might be wondering why we need to bring the parent's `visit` methods into scope. Aren’t they already present due to the public superclass declaration? Well,
  - If you overload a method that you are inheriting, so your parent gives you a method with one signature, and you give the same method with another signature, they are not considered equivalent. 
  - In terms of overload resolution, the one in your scope will take priority over anything else. 
  - So in order to make an inherited method that is an overload of a method that you have, operate at the same level of preference for overload resolution, you have to use a `using` to bring it into your scope.
  - That's true whether you are writing templates, or you are writing ordinary classes.

Anyway, now, what we can do is:

_BookVisitor.h_
```C++
class Book; class Text; class Comic; // forward declaration

// this will generate the old BookVisitor
using BookVisitor = Visitor<Book, Text, Comic>;
```

---
[Abstraction over Iterators <<](./problem_25.md) | [**Home**](../README.md) | [>> I want an even faster vector](./problem_27.md)
