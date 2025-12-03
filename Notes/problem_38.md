[Generalize the Visitor Pattern! Part 2! <<](./problem_33.md) | [**Home**](../README.md) | [>> Total Control](./problem_35.md) 
# Problem 37: Policies
## **2025-11-26**

Recall - [Problem 10](Notes/problem_10.md) talked about how we can make accessing array safe for `vector`.
- We offered them a choice, either using `[]` (unchecked) or using `.at()` for bound checking.
- What if the user wants to check access using `[]`?

Another way to solve the problem:
- Make the choice of checked/unchecked once, and have it be part of the type of the vector class.

Can solve this with CRTP. Create 2 CRTP superclasses, each with a different implementation of operator `[]`.
- These are called **Policy Classes**.

```C++
template <typename T, typename V>
class Unchecked {
    T& operator[](size_t n) {
        return static_cast<V*>(this)->theVector[n];
    }
}

template <typename T, typename V>
class Checked {
    T& operator[](size_t n) {
        V* sub = static_cast<V*>(this);
        if (n >= sub->n) throw range_error;
        return sub->theVector[n];
    }
}

template <typename T, template <typename, typename> Policy>
class vector : public Policy<T, vector<T>> {
    size_t n, cap;
    T* theVector;
    friend class Policy<T, vector<T>>;
public:
    // ...
}
```

- `vector` is parameterized by an uninstantiated template `Policy`.
- Also knowns as **Template Template Parameter**.

```C++
int main() {
    vector<int, Unchecked> v{1, 2, 3};
    vector<int, Checked> w{4, 5, 6};
    v[3]; // UB
    w[3]; // throws
}
```


---
[Generalize the Visitor Pattern! Part 2! <<](./problem_33.md) | [**Home**](../README.md) | [>> Total Control](./problem_35.md) 
