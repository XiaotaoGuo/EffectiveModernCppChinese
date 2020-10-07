# Use `std::move` on rvalue references, `std::forward` on universal references

右值引用只用于绑定哪些需要移动的对象。如果你有一个右值引用参数，这意味着你*清楚*那个对象注定是要被移动的。

```cpp
class Widget {
  Widget(Widget&& rhs) // rhs definitely refers to an object eligible for moving
  …
};
```

也就是说，当你将这些对象传进其他函数时，你允许那些函数利用这些对象的右值性质。而实现这个想法的方法是通过将绑定这些对象对象的参数转换为右值。正如
Item 23 中的解释，这不仅是 `std::move` 要做的事情，更是这个特性被设计的用意：

```cpp
class Widget {
public:
  Widget(Widget&& rhs)              // rhs is rvalue reference
    : name(std::move(rhs.name)),
      p(std::move(rhs.p))
      { … }
    …
};
```

相对而言，一个通用引用（详见 Item 24），*也可以*
绑定一个可用于移动的对象。只用当利用右值来初始化通用引用时才应该将他们转换为右值。 Item 23 中也解释了这正是
`std::forward` 要做的：

```cpp
class Widget {
public:
  template<typename T>
  void setName(T&& newName)             // newName is unversal reference
  { name = std::forward<T>(newName); }
};
```


