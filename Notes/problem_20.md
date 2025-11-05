[Abstraction over containers << ](./problem_18.md) | [**Home**](../README.md) | [>> I'm leaking!](./problem_20.md)

# Problem 20 - What's better than one iterator?
## **2025-10-21**
- Two iterators!

Consider `Vector<T>::erase`:
- Takes an iterator to the item.
- Removes the item.
- Shuffle the following items backward.
- Returns an iterator to the point of erasure.

O(n) cost for shuffling, fine.

But
- What if you need to erase several consecutive items?
- Call `erase` repeatedly? Pay that O(n) cost multiple times.

Faster - Shuffle the items up `k` positions (in one step each) where `k` is the number of items being erased.

For thsi reason, we should make a `range` version of `erase`.

```C++
iterator Vector<T>::erase(iterator first, iterator last);
```

- Erases items in `[first,last)` and only pays the linear cost once.

So maybe iterator isn't the right level abstraction.
- Maybe the right abstraction encapsulates a pair of iterators - a range.

Consider: Compusing multiple functions on some input.

Ex.
Filter and transform (say take all the odd numbers and square them).

```C++
auto add = [](int n) {return n%2 != 0; };
auto sqr = [](int n){return n*n;};

Vector v {...};
Vector<int> w (v.size()), x(v.size());
copyIf(v.begin(), v.end(), w.begin(), add);
transform(w.begin(), w.end(), x.begin(), sqr);
```

2 Problems:
1. The calls don't compose well. Need 2 separate function calls. No way to chain them.
2. Needed to create a `Vector` for intermediate storage.
These functions return an iterator to the beginning of the output range.

What if instead we got a pair of iterators to the beginning and end of the output range.

```C++
template<typename T> class Range {
    Iter start, finish;
    public:
    Iter begin() { return start; }
    Iter end() { return finish; }
};
```

And what if, instead, `copyIf` and `transform` took a `Range`, rather than a pair of iterators.

```C++
transform(copyIf(v, odd), sqr); // v is a `Range`. A range is anything with begin() and end() producing iterator types. So `Vector` is a range.
```

Functions now composable. What about intermediate storage?

Who says we need it now?

A range only needs to look like it's sitting on top of a container.

Instead: Have the `Range` fetch items on demand.

Filter - On fetch - Iterate through the `Range` until you find an item that satisfies the predicate. Then return it.

Transform - On fetch - Fetch an item `x` from the `Range` below. Then return `f(x)`.

These `Range` objects are called views. On-demand fetching, no intermediate storage.

These exists in the C++20 standard library.

```C++
import <ranges>;
vector v {1,2,3,4,5,6,7,8,9,10};
auto x = std::ranges::views::transform(
            std::ranges::views::filter(v, odd), sqr
        );
```

It gets better: `filter`, `transform` take a second form:

```C++
filter(pred), transform(f) // Just supply the function, not the range
```

Become callable objects, parameterized by a range `R`.

Ex.
```C++
filter(pred)(R), transform(f)(R)
```

So our example becomes:

```C++
transform(sqr)(filter(odd)(R))
```

Then operator `|` is defined (Normally the bitwize or operator) so that `R|A` is mapped to `A(R)`. So `B(A(R))` is `B(R|A)` and hence `R|A|B`.

So we can write

```C++
auto x = v 
            | std::ranges::views::filter(odd) 
            | std::ranges::views::transform(sqr);
```

- Just like a Bash pipeline!

---
[Abstraction over containers << ](./problem_18.md) | [**Home**](../README.md) | [>> I'm leaking!](./problem_20.md)
