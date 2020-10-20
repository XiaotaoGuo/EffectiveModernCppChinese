# Use `std::move` on rvalue references, `std::forward` on universal references

右值引用只用于绑定哪些需要移动的对象。如果你有一个右值引用参数，这意味着你*清楚*那个对象注定是要被移动的。

```cpp
class Widget {
  Widget(Widget&& rhs) // rhs definitely refers to an object eligible for moving
  …
};
```

也就是说，当你将这些对象传进其他函数时，你允许那些函数利用这些对象的右值性质。而实现这个想法的方法是通过将绑定这些对象对象的参数转换为右值。正如 Item 23 中的解释，这不仅是 `std::move` 要做的事情，更是这个特性被设计出来的目的：

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

相对而言，一个通用引用（详见 Item 24），*也可能*绑定一个可用于移动的对象。而通用引用只有当在通过右值来初始化时才应该被转换为右值。 Item 23 中也解释了这正是`std::forward` 要做的：

```cpp
class Widget {
public:
  template<typename T>
  void setName(T&& newName)             // newName is unversal reference
  { name = std::forward<T>(newName); }
};
```

简而言之，当右值引用被转发至其他函数时，应该*无条件*被转换成右值（通过`std::move`），因为他们总是与右值绑定。而对于通用引用，当他们被转发至其他函数时应该*有条件*地被转换转换成右值（通过`std::move`），因为他们只是*有时候*绑定右值。

Item23 解释了对右值引用使用 `std::forward` 有可能会作出合适的行为，但是源代码会显得很罗嗦，容易造成误解并且让人不习惯，所以你应该避免对右值引用使用 `std::forward`。而更糟糕的做法是对通用引用使用 `std::move`，因为它有可能造成对左值（例如局部变量）作出预想之外的改变：

```cpp
class Widget {
public:
  template<typename T>
  void setName(T&& newName)       // universal reference
  { name = std::move(newName); }  // compiles, but is bad, bad, bad
  ...

private:
  std::string name;
  std::shared_ptr<SomeDataStructure> p;
};

std::string getWidgetName();      // factory function

Widget w;

auto n = getWidgetName();         // n is local variable

w.setName(n);                     // moves n into w! n's value now unknown

...
```

在这里，局部变量 `n` 被传递给 `w.setName`，调用者假定了 `w.setName` 对 `n` 进行的是只读操作，这是可以理解的。但因为 `setName` 内部使用了 `std::move` 将它的引用参数无条件转换成了右值，所以 `n` 的值被移动到了 `w.name`，而 `n` 在 `setName` 调用结束之后其内部值不确定。这种行为有可能会造成调用者绝望——甚至诉诸暴力。

你有可能会认为 `setName` 本身不应该将它的参数申明为通用引用。这样的应用不能声明为 `const` （见 item24），但 `setName` 当然不应该修改它的参数。你可能会指出如果 `setName` 如果为了 `const` 左值以及（非 `const`）右值进行了重载，整个问题都可以被避免。像这样：

```cpp
class Widget {
public:
  void setName(const std::string& newName)   // set from const lvalue
  { name = std::move(newName); }  
  
  void setName(std::string&& newName)        // set from rvalue
  { name = std::move(newName); }

  ...
};
```

这种方式在这个例子当然可行，但会有一些缺点。首先，需要编写和维护更多的代码（使用了两个函数而不是一个模板）。因此，这种方法有可能会更低效。举个例子，考虑以下 `setName` 的用法：

```cpp
w.setName("Adela Novak");
```

如果采用使用通用引用参数版本的 `setName`，字符串字面量 `"Adela Novak"` 会被传递给 `setName`，并且会进一步传给 `w` 中 `std::string` 中的赋值操作符（assignment operator）。因而 `w` 中的 `name` 成员数据可以直接用字面量进行赋值，不会有临时的 `std::string` 对象被构造。然而，如果采用重载版本的 `setName`，一个临时的 `std::string` 对象会被构造出来并且绑定到 `setName` 的参数，然后这个临时的 `std::string` 会被移动到 `w` 的成员数据。因此，对 `setName` 的一次调用会执行一次 `std::string` 的构造函数（用来构造临时变量），一次 `std::string` 的移动赋值操作（来将 `newName` 移动至 `w.name`）以及一次 `std::string` 的析构参数（来清理临时变量）。大部分情况下，想对于仅仅使用一个 `const char*` 指针来调用 `std::string` 的赋值操作符，这种做法显然是一个更加昂贵的操作序列。额外的开销有可能会根据不同实现方式有区别，并且这些额外开销值不值得担心也是根据不同应用或者不同库的情况具体分析，但事实是通过一对左值引用和右值引用重载函数的方式来代替一个使用模板的通用引用很游客能在大多数情况下会造成运行时的开销。如果我们泛化这个例子，考虑 `Widget` 的成员变量有可能是某个随机的类型（而不是知道它是 `std::string`），这种性能差别有可能会显著变大，因为不是所有类型的移动开销都像 `std::string` 这么低（见 item29）。

然而，对左值和右值进行重载最严重的问题不是源代码的体积或者可读性，也不是代码的运行性能，而是代码的可扩展性太差。`Widget::setName` 仅仅接收一个参数，因此只需要两个重载函数。但对于那些接收更多的参数的函数，其中每一个参数都有可能是左值或者右值，需要重载的数量会几何增加，`n` 个参数会需要 `2^n` 个重载。这还不是最差情况，有一些参数，实际上就是函数模板，有会接收*不限数量*的参数，每个参数都有可能是左值或者右值。这一类函数典型的例子是 `std::make_shared` 和 （对于 C++14） `std::make_unique` （见 item21）。他们最常被使用的重载函数声明如下：

```cpp
template<class T, class... Args>
shared_ptr<T> make_shared(Args&&... args);    // from C++11 Standard

template<class T, class... Args>
unique_ptr<T> make_unique(Args&&... args);    // from C++14 Standard
```

对于像这样的函数，对左值和右值进行重载不可行，唯一的方法是使用通用指针。并且在这样的函数内部，我可以像你保证 当参数被传递到其他函数时肯定是使用 `std::forward`，这也是你应该要做的。

当然，通常，最终（不一定最初）在某些情况下，在一个函数里面，你会想要不止一次使用绑定在右值引用或者通用引用的对象。你想要保证他不会被移动直到你已经完成对它的使用。在这种情况下，你会想要只有在*最后一次*使用这个引用时才引用 `std::move` （对右值引用）或者 `std::forward` （对通用引用）。举个例子：

```cpp
template<typename T>
void setSignText(T&& text)                      // text is univ. reference
{
  sign.setText(text);

  auto now = std::chrono::system_clock::now();  // use text, but don't modify it

  signHistory.add(now, std::forward<T>(text));  // conditionally cast text to rvalue
}
```

这里，我们想要保证 `text` 的值不会被 `sign.setText` 修改，因而我们想要在调用 `signHistory.add` 时使用它。因此，只对最后一次使用该通用引用时使用 `std::forward`。

对于 `std::move` 也是采用同样的思路（即只在最后一次使用右值引用时应用 `std::move`）。但很重要的一点是，有很少数情况下，你会想要调用 `std::move_if_noexcept` 而不是 `std::move`。如果想要学习在何时以及为何要这么做，参考 item14。

如果你是在一个*返回值*的函数中，并且你正在返回一个绑定到右值引用或者通用引用的对象，你会想要在返回引用时应用 `std::move` 或者 `std::forward`。想知道为什么，考虑一个将两个矩阵相加的操作符 `operator+`，其中左侧矩阵已知是个右值（因此可以重用它的空间来放置矩阵的和）：

```cpp
Matrix operator+(Matrix&& lhs, const Matrix& rhs) // by-value return
{
  lhs += rhs;
  return std::move(lhs);                          // move lhs into return value
}
```

通过在 `return` 语句中（通过 `std::move`）将 `lhs` 转换成右值，`lhs` 会被移动到函数返回值的位置中。如果对 `std::move` 的调用被省略：

```cpp
Matrix operator+(Matrix&& lhs, const Matrix& rhs) // as above
{
  lhs += rhs;
  return lhs;                                     // copy lhs into return value
}
```

在这种情况下，`lhs` 是左值，因此会强制令编译器将它*拷贝*进返回值的位置。假设 `Matrix` 类型支持移动构造，移动构造比拷贝构造更加高效，在 `return` 语句中使用 `std::move` 可以令代码更高效。

假如 `Matrix` 不支持移动，把它转换成右值也不会有副作业，因为转换的右值只会被 `Matrix` 的拷贝构造函数进行拷贝（见 item23）。假如 `Matrix` 在之后被修改成支持移动操作，`operator+` 会在它下次被编译的时候自动得到相应的性能提升。也就是说，通过在返回值的函数中对要返回的右值引用应用 `std::move` 没有任何坏处（并且有可能有好处）。

对通用引用和 `std::forward` 的情况也是类似的。考虑一个函数模板 `reduceAndCopy` ，它接收一个可能是 unreduced 的 `Fraction` 对象，然后进行 reduce，并且返回被 reduce 之后的值的副本。假如原始对象是个右值，它的值应该被移动到返回值（从而避免了制造副本的开销），但加入原始变量是一个左值，则必须要创建一个副本。因此：

```cpp
template<typename T>
Fraction reduceAndCopy(T&& frac)  // by-value return universal reference param
{
  frac.reduce();
  return std::forward<T>(frac);   // move rvalue into return value, copy lvalue
}
```

假如不调用 `std::forward`， `frac` 会被无条件拷贝进 `reduceAndCopy` 的返回值。

有一些程序员了解了以上的信息并且尝试将它扩展至有一些不适用的情况。“假如对要拷贝进返回值的右值引用参数使用 `std::move` 可以使拷贝构造变成移动构造”，他们是这么想的，“我也可以对要返回的局部变量进行同样的优化”。也就是说，他们想，给定返回局部变量值的函数，例如：

```cpp
Widget makeWidget() // "Copying" version of makeWidget
{
  Widget w;         // local variable

  ...               // configure w

  return w;         // "copy" w into return value
}
```

他们可以通过将 "拷贝" 变成移动来进行 "优化"：

```cpp
Widget makeWidget()    // Moving version of makeWidget
{
  Widget w;
  ...
  return std::move(w); // move w into return value (don't do this!)
}
```

我故意使用双引号应该可以提醒你这个理由是有缺陷的，但是为什么呢？

他是有缺陷的，因为在这类优化上，标准委员会思考地比这类程序员要更加超前。在很久之前，人们就知道 “拷贝”版本的 `makeWidget` 可以通过在返回值的位置构造局部变量来避免复制局部变量的拷贝。这也被称为*返回值优化*（return value optimization, RVO）。自从有了这样的实现之后，这样的行为就被 C++ 标准允许了。

解释这样的（对上述行为的）同意是一件比较琐碎的事情，因为你会想要在那些不会影响软件能观察到的行为的地方允许这样的*拷贝省略*。用另一种方式来解释标准的合理性的话，这项许可中允许编译器在以下情况在一个返回值的函数中省略对一个局部对象_[1]_的拷贝（或移动）：（1）局部变量的类型和函数返回值的类型一致，（2）该局部变量是函数要返回的对象。在了解这个信息之后，再重新看一下 `makeWidget` 的 “拷贝” 行为：

```cpp
Widget makeWidget() // "Copying" version of makeWidget
{
  Widget w;         // local variable

  ...               // configure w

  return w;         // "copy" w into return value
}
```

（对于拷贝省略的）两个条件在这里都满足，并且我可以跟你保证，对于这段代码，所有好的 C++ 编译器都会采用 RVO 来避免对 `w` 进行拷贝。这意味着这个“拷贝”版本的 `makeWidget` 实际上没有拷贝任何东西。

而移动版本的 `makeWidget` 则做了所有它的名字中指明了它要做的事情（假定 `Widget` 提供了一个移动构造函数）：他将 `w` 的内容移动至 `makeWidget` 的返回值位置。但是为什么编译器不采用 RVO 来省略移动操作，并在为函数返回值分配的内存空间重新构造 `w` 呢？答案很简单：因为它做不到。上述条件（2）规定了 RVO 只能引用在返回值是局部对象的场合，这并不是移动版本的 `makeWidget` 在做的事情。重新看一下它的 `return` 语句：

```cpp
return std::move(w);
```

在这里返回的并不是局部对象 `w`，而是一个对 `w` 的引用，也是 `std::move(w)` 的结果。返回一个对局部对象的引用并不满足 RVO 需要的条件，因此编译器必须将 `w` 移动至函数返回值的位置中。那些尝试通过对局部变量应用 `std::move` 来帮助编译器进行优化的开发人员实际上是限制了他们的编译器能够进行的优化手段！

但 RVO 是一种优化方法。编译器并不被*要求* 省略拷贝和移动操作，即使他们被允许这么做。也许你会头疼并且担心你的编译器因为可以惩罚你使用拷贝操作而这么做。又或者你有足够的洞察力来识别出在某些情况下，编译器很难实现 RVO，例如，在函数中有不同返回路径来返回不同局部变量时。（编译器会不得不生成代码在内存分配给函数返回值的位置构造局部变量，但是，编译器怎么决定构造哪个局部变量呢？）假如是这样的话，你会想要付出想对于拷贝操作，移动操作所需的额外代价来作为保险。因此，你可能还是认为对你要返回的局部变量应用 `std::move` 是合理的，因为在这种情况下你会很清楚的知道你永远不会付出拷贝操作的开销。

在这样的情况下，对局部变量使用 `std::move` *仍然*是糟糕的主意。在标准中许可 RVO 的部分言明了，当 RVO 的条件满足，但编译器选择不进行拷贝忽略操作时，要返回的对象*必须作为右值对待*。也就是说，标准中要求了当 RVO 允许时，针对要返回的局部对象，编译器要么执行拷贝忽略，要么隐式应用 `std::move`。因此在“拷贝”版本的 `makeWidget`，

```cpp
Widget makeWidget() // as before
{
  Widget w;

  ...

  return w;
}
```

编译器要么忽略对 `w` 的拷贝，要么将函数看作是下述函数的等效形式：

```cpp
Widget makeWidget()
{
  Widget w;

  ...

  return std::move(w);    // treat w as rvalue, because no copy elison was performed
}
```

对于值传递参数的函数情况也是类似的。作为函数返回值他们不满足拷贝忽略的条件，但在函数将他们返回的时候，编译器一定将他们作为右值对待。因此，当你的源代码想下面这样：

```cpp
Widget makeWidget(Widget w) // by-value parameter of same type as function's return
{
  ...
  return w;
}
```

编译器会采用和对待下述代码一样的处理方法：

```cpp
Widget makeWidget(Widget w)
{
  ...
  return std::move(w);    // treat w as rvalue
}
```

这意味着如果你在一个返回值的函数对要返回的局部对象使用 `std::move` 时，你并不可以帮助编译器（如果他们不执行拷贝忽略，他们必须将局部对象作为右值），但你当让可以阻碍它们（通过禁止 RVO）。虽然有一些情况对局部变量使用 `std::move` 是合理的（例如，当你将它传递一个函数，并且你知道你再也不会使用它），但是作为符合 RVO 条件或者返回值参数的 `return` 语句的一部分不在这些情况中。

[1]合法的局部变量包括大多数局部变量（例如 `makeWidget` 中的 `w`）以及被临时构造出来作为 `return` 语句的一部份的变量。函数参数不在此列。有一些人在对命名和未命名（及临时的）局部对象 RVO 的应用之间作出区分。他们认为 RVO 是针对未命名对象的，而把对于已命名对象的类似行为称为*命名返回值优化*（*named return value optimization*, NRVO）。

请记住：

- 在最后一次使用右值引用以及通用引用分别使用 `std::move` 和 `std::forward`
- 对于在返回值的函数中要返回的右值引用和通用引用做同样的事情
- 假如局部变量满足返回值优化时永远不要对他们使用 `std::move` 或者 `std::forward`
