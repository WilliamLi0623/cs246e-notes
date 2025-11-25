# Problem 32 - Variants Revisited
## **2025-11-20**

Core idea: Turn `variant<T1, ..., Tn>` into a union with fields of type `T1, ..., Tn`.

```C++
template<typename... Types> union variant_base {};

template<typename First, typename ... Rest> union variant_base {
    First first;
    variant_base<Rest...> rest;
};
    
template<typename... Types> struct variant {
    variant_base<Types... u>;
    size_t index;
}
```

Now we need `get<N>`:
- Return the `N`th item of the variant, if the discriminator is equal to `N`.
- Return type is the type of the `N`th item.
  - Hence why `get<N>` is a template - for each `N`, `get<N>` produces a different return type.

```C++
template<size_t N, typename... Types> constexpr variant_alternative_t<N, variant<Types...>>& get(variant<Types...>& v);
```

Note that `get` take s `size_t N` and a pack of types `Types...`.

`Types` is deducible from the arguments, `N` is not.

Therefore `N` must be supplied rest need not be. Hence `get<N>(v)`.

```C++
template<size_t N, typename Variant> struct variant_alternative;

template<size_t N, typename First, typename... Rest> struct variant_alternative<N, variant<First, Rest...>> : variant_alternative<N-1, variant<Rest...>> {};

template<typename First, typename... Rest> struct variant_alternative<0, variant<First, Rest...>> {
    using type = First;
}

template<size_t N, typename Variant> using variant_alternative_t = typename variant_alternative<N, Variant>::type;

template<size_t N, typename... Types> constexpr variant_alternative_t<N, variant<Types...>>& get(variant<Types...>& v) {
    if (v.index != N) {
        // throw some exception
    } else {
        return get_helper<N>(v.u);
    }
}

template<size_t N, typename... Types> constexpr variant_alternative_t<N, variant<Types...>>& get_helper(variant_base<Types...>& u);

template<size_t N, typename First, typename... Rest> constexpr variant_alternative_t<N, variant<First, Rest...>>& get_helper(variant_base<First, Rest...>& u) {
    return get_helper<N-1>(u.rest);
}

template<typename... Types> constexpr variant_alternative_t<0, variant<Types...>>& get_helper(variant_base<Types...>& u) {
    return u.first;
}
```

Exercise: Try type_based `get`. E.g. `get<int>(v)`.
- Complication: It's an error to call `get` if the requested type occurs more than once in the variant.