# Problem 35 - Resolving Methods Overrides at Compile-Time
## **2025-11-25**

Recall: Template Method Pattern

```C++
class Turtle {
    public:
    void draw() {
        drawHead();
        drawShell();
        drawFeet();
    }
    private:
    // ...
    virtual void drawShell() = 0;
}

class RedTurtle : public Turtle {
    private:
    void drawShell() override {
        // draw red shell
    }
};
```