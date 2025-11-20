
## Move / Forward implementation

`std::move` - First attempt:

```C++
template <typename T> T&& move(T &&x) {
    return static_cast<T&&>(x);
}
```

Doesn't quite work, `T&&` is a universal reference, not rvalue reference. If `T` is deduced to be an lvalue reference, then `T&&` is an lvalue reference.
- Need to make sure `T` is not an lvalue reference.
  - Strip off any reference from `T`.

Correct:

```C++
template<typename T> constexpr std::remove_reference_t<T>&& move(T &&x) {
    return static_cast<std::remove_reference_t<T>&&>(x);
}
```

**Exercise:** write `remove_reference`

**Q:** Can we save some typing and use `auto`?

```C++
template<typename T> auto move(T &&x) { ... }
```

**A:** No! By-value auto throws away reference and outer const
- Recall that `auto x = y` gives x the same type as the **value** of y (i.e, the type of `y` as if it was copied).

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
decltype(-) // returns the type - was declared to have
decltype(var)  // returns the declared type of the variable
decltype(expr) // returns lvalue or rvalue, depending on whether the expr is an lvalue or rvalue

int z;
int &y = z;
decltype(y) x = z;  // x is an int&, auto would only give you int
x = 4;  // Affects z

/* Path/Example 1 */
auto x = z;
x = 4;  // Does not affect z

/* Path/Example 2 */
decltype(z) s = z;  // s is an int
s = 5; // Does not affect z

/* Path/Example 3 */
decltype((z)) r = z;    // r is an int&, since (z) is a ref, since () is an expression
r = t;  // Affects z
```


`decltype(auto)`
- Perform type deduction, like auto, but use the decltype rules.
- i.e. don't throw away references.

```C++
template<typename T> decltype(auto) move(T &&x) {
    return static_cast<std::remove_reference_t<T>&&>(x);
}
```

`std::forward` - First attempt:

```C++
template<typename T> constexpr T&& forward(T &&x) {
    return static_cast<T&&>(x);
}
```

- Doesn't seem right - casting `x` to its own type.

**Reasoning:**
- If `x` is an lvalue, `T&&` is an lvalue reference
- If `x` is an rvalue, `T&&` is an rvalue reference

Doesn't work, `forward` is called on expressions that are lvalues, that may point at rvalues.

```C++
template<typename T> void f(T&& y) { // lvalue/rvalue reference
    ... forward(y) ...  // y is an lvalue
}
```

`forward(y)` will _always_ yield an lvalue reference.

In order to work, `forward` must know what type (including lvalue/rvalue) was deduced for `y`, ie. needs to know `T`.

So `forward<T>(y)` would work.

Can we prevent automatic type deduction for `forward`?

```C++
template<typename T> constexpr T&& forward(std::removed_reference_t<T>&x) noexcept {
    return static_cast<T&&>(x);
}
```
