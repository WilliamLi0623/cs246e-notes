# Problem 32 - Variants Revisited
## **2025-11-20**

Core idea: Turn `variant<T1, ..., Tn>` into a union with fields of type `T1, ..., Tn`.

```C++
template<typename... Types> union variant_base {};
template<typename First, typename ... Rest> union variant_base {
    First first;
    variant_base<Rest...> rest;
};
```