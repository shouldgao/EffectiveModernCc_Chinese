我相信我们会同意使用STL容器是一个好主意，我希望Item 18会说服你使用std::unique_ptr是一个好主意，但是我猜测，我们都不喜欢编写std::unique_ptr<std::unordered_map<std::string, std::string>>这样的类型超过一次。光是想想就可能会增加患腕管综合症的风险。

避免这样的医疗悲剧很容易。引进一个typedef:
```cpp
    typedef
        std::unique_ptr<std::unordered_map<std::string, std::string>>
        UPtrMapSS;
```
但是typedefs实在是太C++98了。当然，它们在C++11中也能使用，但是c++11也提供了alias declarations：
```cpp
    using UPtrMapSS =
        std::unique_ptr<std::unordered_map<std::string, std::string>>;
```
考虑到typedef和alias declaration做的是完全相同的事情，怀疑是否有一个可靠的技术理由来选择一个而不是另一个就显得很合理了。

有，但是在那之前，我想先提一下，许多人发现在处理涉及函数指针的类型时，alias declaration更容易理解：
```cpp
    // FP is a synonym for a pointer to a function taking an int and
    // a const std::string& and returning nothing
    typedef void (*FP)(int, const std::string&);     // typedef

    // same meaning as above
    using FP = void (*)(int, const std::string&);    // alias
                                                     // declaration
```
当然，这两种形式都不是特别容易克服的，而且很少有人会花大量时间处理函数指针类型的同义词，因此，这并不是选择alias declaration而不是typedefs的充分理由。

但一个令人信服的理由确实存在:模板。特别是，alias declaration可以模板化(在这种情况下，它们被称为alias templates)，但是typedefs不能。这为c++ 11程序员提供了一种简单的机制，用来表达在c++98中只能把typedef嵌套进模板化的struct才能表达的东西。例如，考虑为使用定制分配器——MyAlloc的链表定义一个同义词。使用alias declaration，这轻而易举：
```cpp
    template<typename T>                            // MyAllocList<T>
    using MyAllocList = std::list<T, MyAlloc<T>>;   // is synonym for
                                                    // std::list<T,
                                                    // MyAlloc<T>>
    MyAllocList<Widget> lw;                         // client code
```
使用typedef，你几乎必须从头开始创建蛋糕：
```cpp
    template<typename T>                             // MyAllocList<T>::type
    struct MyAllocList {                             // is synonym for
        typedef std::list<T, MyAlloc<T>> type;       // std::list<T,
    };                                               // MyAlloc<T>>
    MyAllocList<Widget>::type lw;                    // client code
```
它变得更糟，如果你想在模板中使用typedef来创建一个链表，而链表中保存着由模板形参决定来类型的对象，你必须在typedef名之前加上typename：
```cpp
    template<typename T>
    class Widget {                                 // Widget<T> contains
    private:                                       // a MyAllocList<T>
        typename MyAllocList<T>::type list;        // as a data member
        …
    };
```
这里，MyAllocList<T>::type引用依赖于模板类型参数(T)的类型。因此，MyAllocList<T>::type是一个从属类型。c++的许多讨人喜欢的规则之一就是从属类型的名称前必须加上type name。

如果MyAllocList被定义为alias template，这种对typename的需求消失了(就像麻烦的“::type”后缀一样):
```cpp
    template<typename T>
    using MyAllocList = std::list<T, MyAlloc<T>>;           // as before
    
    template<typename T>
    class Widget {
    private:
        MyAllocList<T> list;                                // no "typename",
         …                                                  // no "::type"
    };
```
对你来说，MyAllocList<T>(即使用alias template)可能看起来就像依赖于模板参数T的MyAllocList<T>::type(即使用typedef)一样，但你不是一个编译器。当编译器处理Widget模板并遇到使用MyAllocList<T>(即使用alias template)，他们知道MyAllocList<T>是一个类型的名称，因为MyAllocList是一个alias template：它必须命名一个类型。因此，MyAllocList<T>是一个独立类型，不需要也不允许使用typename说明符。

相反地，编译器在Widget模板内部看到MyAllocList<T>::type(即使用嵌套的typedef)时，他们不能确定它是否命名了一种类型，因为当MyAllocList<T>::type指涉的不是一个类型而是其它东西时，那儿可能有一个他们还没有看到的MyAllocList的特化版本。这听起来很疯狂，但是不要因为这种可能性而责怪编译器，是知道这种可能性的人类制造了这样的代码。

例如，一些被误导的灵魂可能调制了这样的东西:
```cpp
    class Wine { … };
    
    template<>                                              // MyAllocList specialization
    class MyAllocList<Wine> {                               // for when T is Wine
    private:
        enum class WineType                                 // see Item 10 for info on
        { White, Red, Rose };                               // "enum class"
    
        WineType type;                                      // in this class, type is
        …                                                   // a data member!
    };
```
可以看到，MyAllocList<Wine>::type并不指涉一个类型。如果Widget要用Wine实例化，在Widget模板内的MyAllocList<T>::type将指涉一个成员数据成员，而不是一个类型。因此，在Widget模板内部，无论是否MyAllocList<T>::type指涉一个依附于T的类型，编译器都坚持认为它是一个类型并在它前面加上typename。

如果您曾经做过任何模板元编程(TMP)，那么几乎可以肯定地说，您需要获取模板类型参数并根据它们创建修改过的类型。例如，给定某个类型T，您可能希望去掉T包含的任何常量或者引用限定词，例如，您可能希望将const std:: string&std::string转换成std::string。或者，您可能希望将const添加到一个类型或将其转换为lvalue引用，例如，将Widget转换为const Widget或转换为widget&。（如果你没有做过任何TMP，那就太糟糕了，因为如果你想成为一个真正高效的c++程序员，你至少需要熟悉c++这方面的基础知识，你可以在Item 23和Item 27中看到TMP的实际示例，包括我刚才提到的类型转换这种的。）










