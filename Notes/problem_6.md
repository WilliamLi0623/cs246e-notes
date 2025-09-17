[The Copier is broken! <<](./problem_5.md) | [**Home**](../README.md) | [>> I want a constant vector](./problem_7.md)

# Problem 6: Thievery
## **2025-09-16**

Consider:

```C++
Node oddOrEvens() {
    Node odds {1, new Node{3, new Node{5, nullptr}}};
    Node evens {2, new Node{4, new Node{6, nullptr}}};
    char c;
    cin >> c;
    if (c == '0') return evens;
    else return odds;
}

Node n = oddOrEvens(); // copy constructor, what is "other"? - reference to what?
```

- Temporary object is created to hold the result of `oddOrEvens()`.
- `other` is a reference to this temporary.
- Copy constructor deep-copies the data from this temporary.

**But** 
- The temporary is just going to be thrown out anyway, as soon as the statement `Node m = oddOrEvens();` is done.
- It's wasteful to deep copy the temp.
- Why not steal the data instead? Saves the cost of a copy.
- We need to be able to tell whether "other" is a reference to a temporary object, or a standalone object.

### **Rvalue references** 
- `Node &&` is a reference to a temporary object (rvalue) of type `Node`. We need a version of the constructor that takes a `Node &&`.
- In other words, we will need to be able to tell whether "other" is a reference to a temporary object (rvalue), or a standalone object (lvalue).
- We will do this by overloading the operator based on lvalue or rvalue status.

**Move Constructors** - Its job is to steal other's data when constructing a new object.

```C++
struct Node {
    // ...
    Node(Node &&other): data{other.data}, next{other.next} {
        other.next = nullptr; // making sure other does not have anything left
    }
};
```

Similarly:

```C++
Node m;
m = oddOrEvens();  // assignment from temporary
```

Why can't we just swap memory instead of copying field-by-field?
- You can't swap because you don't have an object yet (remember this is currently constructing the object). 
- If you swap, you will be swapping garbage to `other`.
- Even when you are taking data from other object, you need to make sure it is still in a valid state because it's still an object, its constructor still runs, and if it's invalid it will crash.

**Move assignment operator**

```C++
struct Node {
    ...
    Node &operator=(Node &&other) { 
        using std::swap;            // steal other's data
        swap(data, other.data);     // you are gonna be destroyed anyway
        swap(next, other.next);     // can you take the garbage for me
        return *this;               // Easy: swap without the copy
    }
};
```

Can combine copy/move assignment:

```C++
struct Node {
    ...
    Node &operator=(Node other) { 
        swap(other); // copies if `other` is a lvalue (so it will be destroyed when out of scope)
        return *this;
    }
};
```

- Unified assignment operator (`&=operator(Node other)`).
- Pass by value:
    - Invokes copy constructor if an argument is a lvalue.
    - Invokes move constructor (if there is one) if an argument is a rvalue.
- We still need to implement our move constructor.

**Note:** Copy/swap can be expensive, hand-coded operator may do less copying.

But now consider:

```C++
struct Student {
    std::string name; // string is a class
    Student (const std::string &name): name{name} {  // copies name into field (uses copy constructor)
        // ...
    }
};
// ...
Student Bob{"bob"}; // we are deep copying from a temporary memory (rvalue, we want to steal)
```

What if `name` points to an rvalue?

```C++
struct Student {
    std::string name;
    // ...
    Student (std::string name): name{name} {  
        // {name} may come from an rvalue, but it can be an lvalue
        // in other words, name may refer to an rvalue but name itself is an lvalue
        // went from guaranteed 1 copy to maybe 1 or 2, bad
    }
};
```

- Will copy if `name` is an lvalue, moves if `name` is an rvalue.
- The problem is that, `name{name}` will always be making a copy, because `name` pass-by-value is based on `{name}`, and not what `{name}` actually is. What this means is, even though `{name}` can be a rvalue, the variable itself is an lvalue so it will always make a copy (thats why we have 1 or 2) and if we can remove that copy, we would get what we want (0 copy if rvalue and 1 copy if lvalue).

```C++
struct Student {
    std::string name;
    Student(std::string name): name{std::move(name)} {
        // ...
    }
};
```

`name{std::move(name)}` forces lvalue (`name`) to be treated as an rvalue, now strings move constructor

```C++
struct Student {
    ...
    Student(Student &&other): //move constructor
        name{std::move(other.name)} {}

    Student &operator=(Student other) { // unified assignment
        name = std::move(other.name); // invokes string's move assignment
        return *this;
    }
}
```

If you don't define move operations, copy operations will be used.

If you do define them, then replace copy operations whenever the arg is a temporary (rvalue).

### **Copy/Move Elision**

```C++
vector makeAVector() {
    return vector{}; // Basic constructor
}

vector v = makeAVector(); // Move constructor? Copy constructor?
```

Try in `g++`, just the basic constructor runs, not copy/move. This can be done by using `cout` in the copy/move constructors.

In some circumstances, the compiler is allowed to skip calling the copy/move constructors (but doesn't have to). `makeAVector()` writes its result directly into the space occupied by `v`, rather than copy/move it later.

For C++17, many elision opportunities become mandatory.

```C++
vector v = vector{};    // Formally a basic construction and a copy/move construction
                        // vector{} is a basic constructor
                        // Here though, the compiler is *required* to elide the copy/move
                        // So basic constructor here only 

vector v = vector{vector{vector{}}};    // Still one basic constructor only
```

``` C++
void doSomething(vector v) {...};   // Pass-by-value - copy/move constructor

doSomething(makeAVector());
```

Result of `makeAVector()` written directly into the paramameter, no copy/move constructor called.

So does that mean copy/move constructors never run?

No, they do run, just not as often as you might expect.

If you really need all of the constructors to run:

`g++14 -fno_elide_constructors ...`

**Note:** while possible, can slow down your program considerably

- Copying is an expensive procedure, the compiler skips these constructors as an optimization.

C++ value categories:
```
         glvalue      rvalue
        /       \    /      \
     lvalue     xvalue    prvalue
```

- Glvalue (generalized lvalue):
  - Denotes a storage location.
  - Includes lvalues of xvalues.
- Lvalue (locator value):
    - Can be on the LHS of an assignment.
    - Variables (including constants), field references, array lookups, pointer dereferences, etc.
- Rvalue (read value):
    - Can't be on the LHS of an assignment.
    - Address cannot be taken.
    Includes xvalues and prvalues.
- Prvalue (pure rvalue):
    - Do not denote storage locations.
    - Literals, arithmetic expressions, function calls with non-reference return types, etc.
    - Most rvalues you would come up with are prvalues.
- Xvalue (expiring value):
    - Considered both glvalues and rvalues.
    - Can't take address of an xvalue, but it does denote a storage location.
    - Lvalue constructions on rvalues.
```

Consider:

```C++
Student {"name", ..., ...}.name;
```

Here `Student {"name", ..., ...}` is a prvalue, but `Student {"name", ..., ...}.name` is an xvalue.

- The rule is: prvalues don't generate temporaries.
- Only xvalues generate temporaries, so only xvalues get moved from & only xvalues & lvalues get copied from.

Before C++17, copy/move elision was optional. Now it is required since prvalues don't generate temporaries. Are there cases where ellision is still optional?

Consider:

```C++
vector f() {
    ...
    return makeAVector(); // prvalue, no temporary, no copy/move
}
```

Now consider:

```C++
vector g() {
    vector v = makeAVector(); // prvalue, no temporary, no copy/move
    ...
    return v;                 // not a prvalue
}
```

Could `v` be ellided? Yes - compiler would create `v` in the caller's stack frame when the result would go. But `v` is not a prvalue, so this is not required.

Now consider:

```C++
vector h() {
    vector v = makeAVector(); // prvalue, no temporary, no copy/move
    vecotor w = makeAVector(); // prvalue, no temporary, no copy/move
    ...
    if (...) return v;
    else return w;
}
```

Which of `v, w` would we put in the caller's stack frame to avoid a move? No way to know, so ellision may not be possible.

In this situation - Named Return Value Optimization (NRVO) - ellision is allowed, but not required.

In summary: Rule of 5 (Big 5)

- If you need to customize any one of
  1. Copy constructor
  2. Copy assignment
  3. Destructor
  4. Move constructor
  5. Move assignment

then you usually need to customize all 5.

---
[The Copier is broken! <<](./problem_5.md) | [**Home**](../README.md) | [>> I want a constant vector](./problem_7.md)
