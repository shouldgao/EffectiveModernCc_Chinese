我相信我们会同意使用STL容器是一个好主意，我希望Item 18会说服你使用std::unique_ptr是一个好主意，但是我猜测，我们都不喜欢编写std::unique_ptr<std::unordered_map<std::string, std::string>>这样的类型超过一次。光是想想就可能会增加患腕管综合症的风险。

避免这样的医疗悲剧很容易。引进一个typedef:
```cpp
    typedef
        std::unique_ptr<std::unordered_map<std::string, std::string>>
        UPtrMapSS;
```
但是typedefs实在是太C++98了。当然，它们在C++11中也能使用，但是c++11提供了alias declarations：
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
                                                    // std::list<T,MyAlloc<T>>
                                               
    MyAllocList<Widget> lw;                         // client code
```
使用typedef，你几乎必须从头开始创建蛋糕：
```cpp
    template<typename T>                             // MyAllocList<T>::type
    struct MyAllocList {                             // is synonym for
        typedef std::list<T, MyAlloc<T>> type;       // std::list<T, MyAlloc<T>>
    };                                               
    MyAllocList<Widget>::type lw;                    // client code
```
这变得更糟，如果你想在模板中使用typedef来创建一个链表，而链表中保存着由模板形参来决定类型的对象，你必须在typedef名之前加上typename：
```cpp
    template<typename T>
    class Widget {                                 // Widget<T> contains
    private:                                       // a MyAllocList<T>
        typename MyAllocList<T>::type list;        // as a data member
        …
    };
```
这里，MyAllocList\<T\>::type指涉一个依赖于模板类型参数(T)的类型。因此，MyAllocList\<T\>::type是一个从属类型。c++的许多讨人喜欢的规则之一就是从属类型的名称前必须加上typename。

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
对你来说，MyAllocList\<T\>（即使用alias template）可能看起来就像依赖于模板参数T的MyAllocList<T>::type（即使用嵌套的typedef）一样，但你不是一个编译器。当编译器处理Widget模板并遇到使用MyAllocList\<T\>（即使用alias template），他们知道MyAllocList\<T\>是一个类型的名称，因为MyAllocList是一个alias template：它必须命名一个类型。因此，MyAllocList\<T\>是一个non-dependent类型，不需要也不允许使用typename说明符。

另一方面，编译器在Widget模板内部看到MyAllocList\<T\>::type（即使用嵌套的typedef）时，他们不能确定它是否代表一种类型，因为可能在一个他们还没有看到的MyAllocList的特化版本中，MyAllocList\<T\>::type指涉的不是一个类型而是其它东西。这听起来很疯狂，但是不要因为这种可能性而责怪编译器，是了解这种可能性的人类制造了这样的代码。

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
如你所见，MyAllocList\<Wine\>::type并不指涉一个类型。如果Widget用Wine实例化，在Widget模板内的MyAllocList\<T>\::type将指涉一个成员数据，而不是一个类型。因此，在Widget模板内部，无论MyAllocList\<T\>::type是否指涉一个忠实地依附于T的类型，这也就是编译器们都坚持主张———通过在前面加上typename使它成为是一个类型——的原因。

如果你曾经做过任何模板元编程(TMP)，那么几乎可以肯定地说，你需要获取模板类型参数并根据它们创建修改过的类型。例如，给定某个类型T，你可能希望去掉T包含的任何常量或者引用限定词，例如，你可能希望将const std:: string&转换成std::string。或者，你可能希望将const添加到一个类型或将其转换为lvalue引用，例如，将Widget转换为const Widget或转换为widget&。（如果你没有做过任何TMP，那就太糟糕了，因为如果你想成为一个真正高效的c++程序员，你至少需要熟悉c++这方面的基础知识，你可以看看实际在用的TMP的一些实例，包括我刚才提到的类型转换这种的，以及在Item 23和Item 27中提到的。）

C++11为你提供了执行这类类型转换的工具，通过type traits的形式，后者是模板的一种细分类，位于头文件\<type_traits\>。那个头文件里有很多type traits，它们中并不是都能进行这种转换，其中有一些确实提供可预测的接口。给定一个你想实施转换的类型T，结果类型变为std::transformation\<T\>::type类型。例如：
```cpp 
    std::remove_const<T>::type              // yields T from const T
    std::remove_reference<T>::type          // yields T from T& and T&&
    std::add_lvalue_reference<T>::type      // yields T& from T
```
这些注释只是总结了这些转换的作用，所以不要太拘泥于字面意思。在项目中使用它们之前，我知道你还需要查阅精确的规范文档。

在这里我的动机并不是要给你一个关于type traits的教程。而是，每次在应用这些转换时应注意结束时写上“::type”。如果将它们应用于模板中的形参类型（在实际代码中通常是这样使用它们的），则还必须在每次使用之前加上typename。这两个语法减速带存在的原因是，在C++11中type traits是通过在模板structs中嵌套typedefs实现的。是的，没错，他们是用类型同义词技术——我试图说明你们不如alias template的那个——实现的。

这是有历史原因的，但我们会跳过它(它很无聊，我保证)，因为标准化委员会姗姗来迟地认识到alias templates是更可行的办法，并且，在C++14中为所有C++11中的类型转换加进了对应的模板。这些别名有一个共同的形式：对于C++11中每个转换std::transformation<T>::type，有一个对应的C++14的alias templates，名为std::transformation_t。例子将阐明我的意思：
```cpp  
    std::remove_const<T>::type              // C++11: const T → T
    std::remove_const_t<T>                  // C++14 equivalent
    std::remove_reference<T>::type          // C++11: T&/T&& → T
    std::remove_reference_t<T>              // C++14 equivalent
    std::add_lvalue_reference<T>::type      // C++11: T → T&
    std::add_lvalue_reference_t<T>          // C++14 equivalent
```
C++11的设计在C++14中仍然有效，但我不知道你为什么要用它们。即使你不能使用C++14，自己编写alias templates也是轻而易举的事情。用到的只是C++11的语言特性，小孩子也能依葫芦画瓢，不是吗？如果你碰巧有电子版本的C++14标准，这更简单，因为需要的只是复制粘贴。来，我来帮你开始：
```cpp   
    template <class T>
    using remove_const_t = typename remove_const<T>::type;
    
    template <class T>
    using remove_reference_t = typename remove_reference<T>::type;
    
    template <class T>
    using add_lvalue_reference_t =
        typename add_lvalue_reference<T>::type;
```
看到了吗?不能更简单了。

**需要记住的事**
+ typedefs不支持模板化，但是alias declarations支持。
+ Alias declarations避免了“::type”后缀，并且，在模板中，前缀“typename”通常需要指涉typedefs。
+ C++14为所有C++11的type traits转换提供alias templates版本。
