[Collecting Stats <<](./problem_29.md) | [**Home**](../README.md) | [>> Polymorphic Cloning](./problem_31.md)

# Problem 34 - Resolving Method Overrides at Compile-Time
## **2025-11-25**

**Recall:** Template Method Pattern

```C++
class Turtle {
    public:
        void draw() {
            drawHead();
            drawShell();
            drawFeet();
        }
    private:
        void drawHead();
        virtual void drawShell() = 0;   // vtable lookup
        void drawFeet();
};

class RedTurtle: public Turtle {
    void drawShell() override;
};
```

- Is there a way to avoid vtable lookup?

**Consider:**

```C++
template<typename T> class Turtle {
    public:
        void draw() {
            drawHead();
            static_cast<T*>(this)->drawShell();
            drawFeet();
        }
    private:
        void drawHead();
        void drawFeet();
};

class RedTurtle: public Turtle<RedTurtle> {
    friend class Turtle;
    void drawShell();
};

class GreenTurtle: public Turtle<GreenTurtle> {
    friend class Turtle;
    void drawShell();
};
```

No virtual method methods, no vtable lookup.
- Drawback: no relationship between `RedTurtle` and `GreenTurtle`
    - Can't store a mix of them in a container.

1. Can give `Turtle` a parent:

```C++
template<typename T> class Turtle: public Enemy { ... };
```

Then can store `RedTurtles` and `GreenTurtles`
- There is no `draw` method in `Enemy`.
- You could give Enemy a virtual `draw` method, but then you have vtables.

```C++
class Enemy {
    public:
        virtual void draw() = 0;
};
```

2. OR - Use a `variant`

---
[Collecting Stats <<](./problem_29.md) | [**Home**](../README.md) | [>> Polymorphic Cloning](./problem_31.md)