[The Middle <<](./problem_17.md) | [**Home**](../README.md) | [>> Abstraction Over Containers](./problem_19.md)

# Problem 18 - A Case Study in Strings
## **2025-10-09**

So if a `String` is a sequence of `char`s, can we just use `Vector<char>` when we want a `String`s?

**Answer:** Yes, in principle. But there might be some reasons not to.

Primarily, `Vector` functions as a container of items that may not have much to do with each others.

But in a `String`, it's the elements taken together as a group that make a `String` what it is.

`String`s are more likely than `Vector`s to:
- Be appended to one another.
- Be compared lexicographically.
- Be searched for a substring.
- Have a substring extracted.

All of these are about characters in the sequence in conjunction with their neighbours.

So, a `String` would likely be expected to support a different interface from `Vector`.

There are different kinds of characters. There is `char` (8 bits), but there are also wider character types.

So `String` should be a template parameterized by character type. (For simplicity, we'll stick with `char` and not write a template.)

No matter what the character type is, they all have one thing in common - they are a basic type, not a class. So no constructors or destructors. No placement new needed.

Simple implementation:

```C++
class String {
    size_t length, cap;
    char *theString;
    // ...
};
```

**Question:** Do we still need null termination?

**Answer:** No, but also kind of. We have a length field now, to tell us how long the string is. And there are two immediate advantages:
1. `length` is O(1)
1. Because we have a length to tell us where the string ends, we can now store `\0`s in the middle of a string.

But:
- We still need to be able to interface with functions expecting `char*`.

```C++
class String {
    // ...
    public:
        // ...
        const char * c_str(); // Returns an equivvalent C-style string (O(1) time)
        // ...
}
```

For this to work, the `char*` must be null-terminated. So we still need `theString[length] = '\0'` even though we don't need this condition to determine the length.

**Note:** If our string does contain `'\0'`s (not as terminates), the `char*` returned by `c_str` will be interpreted as ending at the first `'\0'`.

Comparison (lexicographic) C: `strcmp` - Direct comparison was comparison at pointers.

Since `String` is a class, we can define `operator<`, `operator<=`, etc for strings. 

So instread of `if(strcmp(s1, s2) < 0)`, we can write `if(s1 < s2)` which is more natural. But `strcmp` had one advantage over `operator<`, etc - It is 3-valued.

So we can write:

```C++
char *s1, *s2;
auto result = strcmp(s1, s2); // 1 linear scan
if (result < 0) {...}
else if (result == 0) {...}
else {...}
```

vs.

```C++
String s1, s2;
if (s1 < s2) {...}
else if (s1 == s2) {...} // 2 linear scans, mostly the same operation done twice
else {...}
```

If we want to compete with C, we need a 3-valued comparison operation: `operator<=>` (spaceship operator)

The standard provides a class `std::strong_ordering` and constants `std::strong_ordering::{less, equal, greater}` that we can use as the result of comparisons.

These constants compare, respectively, as `{<, ==, >} 0`.

By default, `<=>` does lexicographic comparison for free, if you ask for it:

```C++
class Vec{
    int x, y;
    public:
        std::strong_ordering operator<=>(const Vec &other) const = default;
}
```

Or we could write it ourselves:

```C++
class Vec {
    // ...
    std::strong_ordering operator<=>(const Vec &other) const {
        auto n = x <=> other.x;
        return (n == 0) ? (y <=> other.y) : n;
    } // equivalent to default
};
```

But for `String`, the default would do pointer comparison on `theString`, so we need to write our own:

```C++
class String {
    // ...
    std::strong_ordering operator<=>(const String &other) const {
        for (size_t i = 0; i < std::min(length, other.length); ++i) {
            if(theString[i] != other.theString[i])
                return (theString[i] <=> other.theString[i]);
        }
        return (length <=> other.length);
    }
};
```

This also gets us `s1 < s2` for free, also `<=, >, >=`. `s1 < s2` is rewritten by the compiler as `(s1 <=> s2) < 0`, etc.

There's a small problem with (in)equality.
- Always does a linear scan, even if the strings have different lengths.
- But strings of different lengths cannot be equal.

**Solution:** Write a specialized `operator==` (exercise).
- Compare lengths first.
- Then do a linear scan.

C++ will use this for both `==` and `!=`.

### Optimizing the representation:

One thing we know about `char`s, vs `T`s - They are small (one byte).
A lot of `String`s are small too.

Consider `String s = "a"`:

```
// TODO
```

Pointer dereference leads to poor locality - Seems wasteful when the payoff is one character.

What if short `String`s were represented differently?

Multiple representations for an object. Let's talk about `union`s.

For types `T1, T2`, consider:

```C++
union U {
    T1 t1;
    T2 t2;
};
```

**WRONG - dangerous**

Consider:

```C++
T1 x;
U u;
u.t1 = x; // OK
T2 y = u.t2; // Undefined behaviour
```

`union`s must be used consistently - You must remember which field of a union is active.

## **2025-10-21**

Typically - discriminator variable, or the union appears inside a struct with a discriminator variable.

Traditional `Vector`/`String` layout:

```C++
class String {
    size_t n;
    size_t cap; // Short String Optimization (SSO)
    char *theString; // Use this space as an array of `char`s, if n is small
}
```

So the size `n` can also be the discriminator.

```C++
class String {
    struct S {
        size_t cap;
        char *theString;
    };
    size_t n; //if n >= sizeof(s) use s; else use arr
    union {
        S s;
        char arr[sizeof(s)];
    };
};
```

Assuming `size_t` is 8 bytes, pointers are 8 bytes. SSO can hold 15 bytes (+1 for '\0').

But (trick):
What if we store `n` last, instead of first? (Note: assumes big-endian architecture)

If the `String` is short, the first 7 bytes of `n` will be zero. So the first byte of `n` could function as the null terminiator. Then we could hold 16 characters and the null terminator.

Extracting substrings:
- `String::substr` creates a new `String` - new heap allocation (or SSO).
- Can we do better? What if we know the substring will not be mutated?

Consier:

```C++
class StringView {
    const char *start;
    size_t n;
};
```

A non-owning slice of a `String`.

Points to a position in an existing `String`, or `char*`, or `Vector<char>`, plus a length (to encode the end).
- No allocation.
- Smaller than `String` - Pass-by-value is reasonable.
- Copy operations are trivial (No Big 5).

What could we do with this? What methods could we give `StringView`?
- Iteration - `begin()`, `end()` - Range-based loops.
- Extracting further substrings. - `remove_prefix`, `remove_suffix`, `substr`. The first two are modifying the endpoints, and the last one creates a new `StringView`.

**Note**: Null termination is not guaranteed, so range-based looping is appropriate.

---
[Insert/Remove in the middle <<](./problem_17.md) | [**Home**](../README.md) | [>> Abstraction Over Containers](./problem_19.md)