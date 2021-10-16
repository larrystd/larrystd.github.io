---
title: modern_cpp_3:右值引用与lambda
date: 2021-09-08
updated: 2021-09-08
tags: 
  - cpp
  - modern cpp
categories:
  - cpp base
aplayer: true
---


## 右值引用, 移动语义, 完美转发
### 条款 23, 理解`std::move`和`std::forward`

* 移动语义使编译器最有可能用廉价的移动操作来代替昂贵的复制操作, 如同复制构造函数和复制赋值操作符赋值对象, 移动构造函数和移动赋值操作具有控制移动语义的权利。此外, 有些类型只可移动(move-only), 例如`std::unique`, `std::future`, `std::thread`。

* 完美转发使接受任意数量参数的函数模板成为可能, 它可以将参数转发到其他的函数, 使目标函数接受的参数和转发函数的参数(左值右值)一致。

* 移动的实现依赖于内部的指针, 例如对象a移动构造对象b来自于a把内部的指针直接给b(显然前提是a不能再用这个指针了, 对于右值来说自然不会用了), 但对于数组等连续开辟内存的对象, 内部并没有指针的, 移动构造的优势可能没有那么大(但不妨碍底层可能用到了指针, 指针的使用在底层如此广泛, 移动的优势依然存在)。

* `std::swap`是移动语义的一种常用实现方案 (显然`swap`也是移动内部指针)。

事实上, `std::move`和`std::forward`仅仅使执行转换cast的函数(也就是函数模板), `std::move`无条件将参数转换为右值`, `std::forward`只在指定情况下满足时转换。

```cpp
//std::move近似的实现

template <typename T>
typename remove_reference<T>::type&& move(T&& param) {
    using ReturnType = typename remove_reference<T>::type&&;
    return static_cast<ReturnType>param;
}
```
以上
* move函数接收一个通用引用(T&&), 返回一个右值引用(`remove_reference<T>::type&&`。

* 使用了萃取的手法, 如果传入的时一个引用(T&), 将`remove_reference<T>`应用到T&上, 萃取出类型T, 保证`remove_reference<T>::type&&`返回值是一个右值引用。

* 最后的`static_cast`将param转化成一个右值
* `std::move`就是把参数转换成一个右值

<!-- more -->

#### 不要对想移动的对象声明为常量

```cpp
class Annoation {
public:
    explicit Annoation(const std::string text) 
        : value(std::move(text))
        {...}
    
    string(const string& ths);  // 拷贝构造函数
    string(string&& rhs);   //移动构造函数
};
```
以上text是赋值到value中而不是拷贝。原因在于text参数类型为const, 然而移动构造函数只能接收非const右值。这样`std::move`将text转化为const右值后,该右值与拷贝构造函数绑定。(**指向常量的左值引用允许绑定到一个常量右值上**)

而且,`std::move`并不保证移动, 它只是复制将参数转换成右值而已。

#### `std::forward`只把右值参数转换成右值

```cpp
void process(const Widget& lvalArg);    // 左值处理
void process(Widget&& rvalArg); // 右值处理

template<typename T>
void logAndProcess(T&& param) {
    auto now = std::chrono::system_clock::now();
    process(std::forward<T>(param));    // 传入的为右值, 转右值
}

Widget w;

logAndProcess(w);   // 传入参数为左值, 调void process(const Widget& lvalArg); 
logAndProcess(std::move(w)); // 传入参数为右值, 调void process(Widget&& rvalArg); 
```

* `std::forward<T>`需要一个模板类型参数, `std::move`不需要
* `std::move`和`std::forward<T>`只在编译期起作用, 运行期什么也不做。
* `std::move`无条件转右值, `std::forward<T>`保留原有的左值或者右值属性。


### 条款24 区分通用引用和右值引用

`T&&`有两种意思, 右值引用, 以及通用引用。

两种情况下会出现通用引用
1. 函数模板参数或者`auto`声明
2. 存在类型推导。

如果没有类型推导, 则是右值引用。
```cpp
template <typename T>
void f(T&& param);  // 通用引用

Widget w;
f(w);   // 传入左值
f(std::move(w)) // 传入右值

template <typename T>
void f(std::vector<int>&& param);   // param是一个右值引用
```

类型声明为`auto&&`是通用引用, 因为等价于模板`T&&`参数的类型推导。
```cpp
auto timeFuncInvcation = 
[](auto&& func, auto&&... params) {
    std::forward<decltype(func)>(func) (
        std::forward<decltype(params)>(params) ...
    );
}
```
以上`auto&&`的`func`和`params`都是通用引用

* 通用引用的类型推导是通过引用折叠reference collapsing实现的
* 通用引用, 如果被右值初始化就会成为右值引用, 被左值初始化会成为左值引用
* type&& 如果可以发生类型推导, 是一个通用引用, 不发生类型推导, 是一个右值引用。

### 条款25 对右值引用使用`std::move`, 对通用引用使用`std::forward`

* 右值引用仅绑定可以移动的对象, 既然这个对象可以移动, 自然需要对对象的车成员变量以右值的形式传递给其他对象

```cpp
class Widget {
public:
    widget(Widget&& rhs) : name(std::move(rhs.name)), p(std::move(rhs.p)) {...} // 右值形式传参

private:
    std::string name;
    std::shared_ptr<Object> p;
}
```

* 基于通用引用的特点, 被右值初始化就会成为右值引用, 被左值初始化会成为左值引用。用`std::forward`更为贴切

```cpp
class Widget {
public:
    template<typename T>
    void setName(T&& newName) {
        name = std::forward<T>(newName);
    }
}
```

lambda表达式
### 避免使用默认捕获模式

cpp11有两种lambda表达式捕获模式, 按引用捕获和按值捕获, 分别表示为`[&]`, `[=]`

* lambda创建的运行时对象是闭包对象, 依赖捕获模式。闭包类是实例化闭包对象的类。

* 按引用捕获使闭包中包含了对局部变量或某个形参的引用, 如果闭包对象生命周期超过了局部变量的生命周期, 闭包中的引用会变成悬空引用。

例如
```cpp
void addDivisorFilter() {
    auto calc1 = computeSomeValue1();
    auto calc2 = computeSomeValue2();
    auto divisor = computeDivisor(calc1, calc2);
    filters.emplace_back([&](int value) {
        return value % divisor == 0;
    });
}
```
以上filter添加了一个lambda表达式对象, 该对象捕获了外界的元素`divisor`, 但`divisor`在`addDisvisorFilter`退出后即被销毁, 之后再从`filter`中调用lambda表达式对象就会出错。


* 按值捕获也不一定解决该问题, 引用有可能按值捕获的只是指针, 有可能指向的对象被析构掉(但这种比捕获引用安全一些)。

* 对象里的lambda表达式默认会捕获this指针

```cpp
void Widget::addFilter() const{
    filters.emplace_back ([=] (int value) {
        return value % divisor == 0;
    });
}

// 会捕获this指针, 等价于如下
void Widget::addFilter() const {
    auto currentObjectPtr = this;

    filters.emplace_back(
        [currentObjectPtr](int value) {
            return value % currentObjectPtr->divisor == 0;
        }
    );
}
```

* 闭包内含有this指针的拷贝, 可能会发生指针悬空,一个方法使做一个局部拷贝

```cpp
void Widget::addFilter() const {
    auto divisorCopy = divisor;
    filters.emplace_back (
        [divisorCopy](int value) {
            return value % divisorCopy == 0;
        }
    );
}
```

* 尽量显示捕获变量, 而不是采用`[],[=],[&]`等默认捕获全部变量。



### 条款 41 参数拷贝和移动开销低，可以总是考虑按值传递

一般的, 为了提高效率, 应该(左值引用)左值, (右值移动)右值。这样对传入的参数, 不论左值右值, 都能高效率。

如下
```cpp
class Widget {
public:
    // 当传入的是左值, 进入左值引用, 传地址
    void addName(const std::string& newName)   {
        /// 这里是将newName拷贝进去, 要调拷贝构造
        names.push_back(newName);
    }
    // 传入的是右值, 执行右值引用函数
    void addName(std::string&& newName) {
        /// 直接将右值移动进names中
        names.push_back(std::move(newName));
    }
    ...
private:
    std::vector<std::string> names;
}
```

还可以借助通用引用, 一个函数实现上述功能

```cpp
class Widget {
public:

    /// 根据传入的左值右值自动转换
    template<typename T>
    void addName(T&& newName) {
        names.push_back(std::forward<T>(newName));
    }
}
```

* 但模板的问题是, 如果参数不同, 编译器会对模板实例成多个函数(每个参数类型都不同),

如果拷贝构造和移动开销低, 可以使用左值拷贝右值移动的办法。

```cpp
class Widget {
public: 
    /// cpp98中传左值右值都是拷贝构造
    /// cpp11中传左值是拷贝构造, 传右值则是移动
    void addName(std::string newName) {
        names.push_back(std::move(newName));
    }
}
```

* 使用第一种左右值重载的方法, 在传参时左右值零开销, 但push_back执行时, 传左值需要一次拷贝, 右值需要一次移动
* 通用模板方式, 可以自适应左值右值, 实际上等价于左值右值(传不同值时模板实际上实例化多个函数), 开销等同于第一种
* 第三种按值传递时, 传左值一次拷贝一次移动, 传右值两次移动, 比重载多了一次拷贝+一次移动。

* 这个条款也就了解下, 还是第一，二种办法比较靠谱

### 条款42 考虑用emplacement代替insert

`std::vector`的`push_back`按左值和右值分别重载
```cpp
template <class T, class Allocator = allocator<T>>
class vector{
public:
    ...
    /// 左值
    void push_back(const &T x);
    /// 右值
    void push_back(T&& );   
}
```

但问题在于, 如果传入的是**类字面量**,例如`"xyz"`, 在push_back过程种会将类字面量重新创建一个string对象, push_back到vector中, 这增加了开销。

* 注意`std::vector<>`只接受拷贝的对象!

* `emplace_back`直接将传递的参数(无论是不是`std::string`)直接传递到`std::vector`的内部, 没有临时变量生成(显然`vector`的内部有相关的实现支持。**内部使用的是完美转发**, 只要没受完美转发的限制。`push_back`显然只是在表层用了两种引用, 内部没有改变。