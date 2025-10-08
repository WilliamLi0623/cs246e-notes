[Is Vector Exception Safe? << ](./problem_16.md) | [**Home**](../README.md) | [>> Abstraction over containers?](./problem_18.md) 

# Problem 17: The Middle
## **2025-10-08**

If we want to insert an item into the middle of a collection - say by index.

For vectors:
- Easy to find the place to insert.
- Shuffling cost (unavoidable).

For lists:
- Finding the insertion point is linear search.
- No shuffling cost. 

Using iterators instead of indices should be about the same for vectors, faster for lists.

```C++
template<typename T> class Vector {
    ...
    public:
        iterator insert(iterator posn, const T &x) {
            ptrdiff_t offset = posn - begin(); // ptrdiff_t incase result is negative (in general)
            increaseCap();
            posn = begin() + offset;
            new(end()) T(std::move_if_noexcept(*(end() - 1)));
            ++n;
            for (iterator it = end() - 2; it != posn; --it) {
                *it = std::move_if_noexcept(*(it - 1));
            }
            *posn = x;
            return posn;
        }
};
```

Exception safe? Assuming `T`'s copy/move operations are exception safe (at least basic guarantee), insert offers the basic guarantee.
- May get a partially shuffled vector, but it will be a valid vector.

**Note:** if you have other iterators pointing at the vector

```
+---+---+---+---+---+  
| 1 | 2 | 3 | 4 |...|  
+---+---+---+---+---+  
  ^       ^   ^    
 it1      h  it2  

and you insert at h  

+---+---+---+---+---+---+  
| 1 | 2 | 5 | 3 | 4 |...|  
+---+---+---+---+---+---+  
  ^       ^   ^    
 it1      h  it2  
```

`it2` will now point at a different item.

**Convention:** after a call to insert or erase, all iterators pointing at or after the point of insertion/erasure are considered invalid and should not be used.
- Similarly, if a reallocation happens, _all_ iterators pointing into the vector become invalid.

Exercises: 
- **erase** - remove the item pointer to by an iterator, return an iterator to the point of erasure
- **emplace** - like insert but takes constructor args

BUT - that means there is a problem with `push_back`. If placement new (constructor) throws and there was a reallocation before that happened, the vector is the same but the iterators were invalidated!

## **2025-10-09**

This is really tricky. A general rule of thumb (in programming only, not during exams) is to deal with the hard stuffs first.
- General technique for exception safety: Do the unsafe things first.
  - If it fails, you have not done anything yet
  - If it succeeds, you are home free :D

So let's write the new item before we transfer the old ones.
```C++
template<typename T> class vector {
    // ...
public:
    void push_back(const T& x) {
        // we will increase size by hand rather than relying on our existing method
        if (vb.n == vb.cap) {
            vector_base<T> vb2{2 * cap};
            new (vb2.v + vb.n) T(x); // if this fails, then vb2 is clean up automatically, which is good
            vb2.n = vb.n + 1;
            try {
                uninitialized_copy_or_move(vb.v, vb.v + vb.n, vb2.v);
                clear();
                std::swap (vb, vb2);
            } catch (...) {
                // if uninit... fails, we still have the new vb2 array, everything else was clean up, and so the only thing left we need to clean up now is the T(x) item we just put in
                (vb2.v + vb2.n - 1)->~T();
                throw;
            }
        } else {
            new (vb.v + vb.n) T(x);
            ++ vb.n; // we don't want to do this postfix, that would be too soon because new (addr) T(x) might fail, and we don't want to increase n if it fails 
        }
    }
}
```
- This offers strong guarantee
- Exercise: go through this again and convince yourself that this is strong guarantee

---
[Is vector exception safe? << ](./problem_16.md) | [**Home**](../README.md) | [>> Abstraction over containers?](./problem_18.md)
