情况是这样的：字面值0是一个int类型而不是指针。如果C++发现一个0，在只能使用指针的上下文中，它会不情愿的把0解释为一个空指针，但是那是迫不得已的情况。C++的基本策略是0是一个int类型，而不是一个指针类型。

实际上，NULL的真实情况也是这样的。NULL的细节上有一些不确定性，因为各个实现被允许给与NULL一个整型（比如说long）而不是int型。这是不寻常的，但这不重要，因为这里的问题不是NULL的准确类型，而是不管是0还是NULL都不是指针类型。

在C++98中，以上导致的一个主要影响是重载指针类型和整型时会导致令人意外的情形。传递0或者NULL给这样的重载时，从不会调用一个指针类型的重载版本。
```cpp
    void f(int);      // f的三个重载
    void f(bool);
    void f(void*);

    f(0);             // 调用f(int), 而不是f(void*)

    f(NULL);          // 可能无法编译, 但一般调用的是f(int)，从不调用f(void*)
```
关于f(NULL)的行为的不确定性是授予给NULL的类型的实现留有余地的一种反映。如果NULL被定义为，比如说0L（也就是说，0是类型long），那么调用是模棱两可的，因为从long到int的转换，long到bool的转换，和0L到void*的转换被认为是完全同等地位的。这个调用的有趣之处在于源代码的表面含义(我是在用空指针——NULL调用f)和它的实际含义(我是在用某种整型——非空指针调用f)之间的矛盾。这种违反直觉的行为导致形成了C++98程序员们的一个准则——避免重载指针和整数类型。那个准则在C++11中仍然有效，因为尽管有本条款的建议，好像还是有一些开发者将继续使用0和NULL，尽管nullptr是一个更好的选择。

nullptr的一个优点是它不是一个整数类型。诚实来说，它也不是一个指针类型，但是你可以认为它是一个所有类型的指针。nullptr的实际类型是std::nullptr_t，并且，在一个非常好的循环定义中，std::nullptr_t被定义为nullptr的类型。类型std::nullptr_t可以隐式地转换为所有原始指针类型，这就是导致nullptr表现的就好像它是所有类型的指针一样的原因。

使用nullptr调用重载函数f会调用void*重载版本(即指针重载版本），因为nullptr不能被视为任何一个整数：
```cpp
    f(nullptr);         // 调用f(void*)重载版本
```
使用nullptr替代0和NULL因而可以避免重载解析中出现的意外，但是这不是它唯一的优点。它可以帮助促进代码更清晰，特别是当使用auto变量的时候。比如，假设你在一个代码库中遇到了这样的代码：
```cpp
    auto result = findRecord( /* arguments */ );
    if (result == 0) {
        …
    }
```
如果您不知道(或者不容易找到)findRecord返回什么，可能就不清楚result是指针类型还是整数类型。毕竟0(变量result测试的值)可以走两种可能。另一方面，如果你看到以下内容，
```cpp
    auto result = findRecord( /* arguments */ );
    if (result == nullptr) {
        …
    }
```
这里没有歧义:result必须是指针类型。

当模板进入画面时，nullptr会特别亮。假设您有一些函数，只有在适当的互斥锁被锁定时才应该被调用。每个函数接受不同类型的指针:
```cpp
    int f1(std::shared_ptr<Widget> spw);         // 只有在适当的互斥锁
    double f2(std::unique_ptr<Widget> upw);      // 被锁定时
    bool f3(Widget* pw);                         // 这些才被调用
```
希望传递空指针的调用代码，可以像这样:
```cpp
    std::mutex f1m, f2m, f3m;                 // mutexes for f1, f2, and f3

    using MuxGuard =                          // C++11 typedef; see Item 9
        std::lock_guard<std::mutex>;
    … 
    {
        MuxGuard g(f1m);                      // lock mutex for f1
        auto result = f1(0);                  // pass 0 as null ptr to f1
    }                                         // unlock mutex

    … 

    {
        MuxGuard g(f2m);                      // lock mutex for f2
        auto result = f2(NULL);               // pass NULL as null ptr to f2
    }                                         // unlock mutex

    … 

    {
        MuxGuard g(f3m);                     // lock mutex for f3
        auto result = f3(nullptr);           // pass nullptr as null ptr to f3
    }                                        // unlock mutex
```
在这段代码的前两个调用中没有使用nullptr是令人遗憾的，但是代码是有效的，这也许对一些东西有用。但是以上代码的重复模式——锁住互斥锁，调用函数，解锁互斥锁——更可悲。这令人感到困扰。这种源代码复制是模板设计要避免的事情之一，所以让我们对这个模式进行模板化:
```cpp
    template<typename FuncType,
             typename MuxType,
             typename PtrType>
    auto lockAndCall(FuncType func,
                     MuxType& mutex,
                     PtrType ptr) -> decltype(func(ptr))
    {
        MuxGuard g(mutex);
        return func(ptr);
    }
```
如果这个函数的返回类型(auto…-> decltype(func(ptr))让你挠头了，帮你的头一个忙，转到Item 3，它解释了发生了什么。你会看到在c++14中，返回类型可以简化为简单的decltype(auto):
```cpp
    template<typename FuncType,
             typename MuxType,
             typename PtrType>
    decltype(auto) lockAndCall(FuncType func,          // C++14
                               MuxType& mutex,
                               PtrType ptr)
    {
        MuxGuard g(mutex);
        return func(ptr);
    }
```
给定lockAndCall模板(任何一个版本)，调用者可以这样写代码:
```cpp
    auto result1 = lockAndCall(f1, f1m, 0);            // error!
    …
    auto result2 = lockAndCall(f2, f2m, NULL);         // error!
    …
    auto result3 = lockAndCall(f3, f3m, nullptr);      // fine
```
当然，他们可以编写代码，但是，正如注释所示，三种情况中的两种，代码都无法编译。第一次调用的问题是，当0被传递给lockAndCall调用时，模板类型推导开始推导其类型。0的类型现在是，曾经是，并且总是int，所以这是参数ptr在调用lockAndCall的实例中的类型。不幸的是，这意味着在lockAndCall内部对func的调用中，传递了一个int，这与f1期望的std::shared_ptr<Widget>参数不兼容。传递给lockAndCall的调用中的0试图表示一个空指针，但实际上传递的是一个普通的int。试图将这个int作为std::shared_ptr<Widget>传递给f1是一个类型错误。使用0调用lockAndCall会失败，因为在模板内部，一个int被传递给一个需要std:: shared_ptr<Widget>的函数。

涉及NULL的调用的分析本质上是相同的。当NULL被传递给lockAndCall时，参数ptr被推断为一个整数类型，当将ptr——int或int-like的类型——传递给f2时，发生类型错误，因为f2希望得到一个std::unique_ptr<Widget>类型。

相反，涉及nullptr的调用没有问题。当nullptr被传递给lockAndCall时，ptr的类型被推断为std::nullptr_t。当ptr传递给f3时，有一个从std::nullptr_t到Widget*的隐式转换，因为std::nullptr_t可以隐式地转换为所有指针类型。

模板类型推导为0和NULL推导出的的“错误”类型（即它们的真正的类型，而不是他们的fallback意义——作为一个空指针的代表）的这个事实，是当您希望引用空指针时，使用nullptr替代0或NULL的最有说服力的理由。使用nullptr，模板不会带来任何特殊的挑战。加上nullptr不受重载解析中意外情形的影响(0和NULL容易受到重载解析意外情形的影响)，结论是牢靠的：当你想指涉一个空指针时，使用nullptr吧，而不是0或者NULL。

**需要记住的事**：
+ 优先使用nullptr，而不是0或者NULL。
+ 避免重载整型和指针类型。
