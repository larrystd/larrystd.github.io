---
title: modern_cpp_1:template
date: 2021-07-22
updated: 2021-07-22
tags: 
  - cpp
  - modern cpp
categories:
  - effective系列
aplayer: true
---

c++98有一套模板类型推导的规则, c++11修改了一些规则并为auto和decltype添加了新规则。

### 条款1 理解模板类型推导

* ParamType是一个指针或引用但不是通用引用

```cpp
template <typename T>
void f(T& param);   // ParamType是一个引用

int x = 27; //int
const int cx = x;1// const int
const int& rx = cx; // rx是指向const int的引用

f(x)    // T为int, param类型为int&
f(cx)   // T是const int, param类型const int&
f(rx)   // T是const int, param类型const int&
// 注意rx类型为引用，但T会推导为一个非引用
```



注意T和param的区别, 这里param是T&

若expr类型是一个引用, 忽略引用部分, 来决定T。也就是说**T会被推导成非引用**。

T和形参匹配得出最终的ParamType, ParamType一般是一个左值引用。

这也是STL模板萃取指针类型的办法, 即参数ParamType为`T*`可以推导出T的类型(因为T被推到成非引用)。

* ParamType是一个通用引用

```cpp
template <typename T>
void f(T&& param);   // ParamType是一个通用引用

int x = 27; //int
const int cx = x;1// const int
const int& rx = cx; // rx是指向const int的引用

f(x)    // x为左值,推到成左值引用 T为int& , param类型为int&
f(cx)   // cx为左值, T是const int&, param类型const int&
f(rx)   // rx为左值 T是const int&, param类型const int&
f(27)   // 27为右值, T为int, param为int &&
```



(ParamType最终被推导成左值引用或右值引用)

如果expr是左值或左值引用, T和ParamType均被推到成左值引用, T&&也是一个左值引用。

如果expr是右值或右值引用，忽略引用部分决定T。也就是说**T会被推导成非引用(类型)**, 而T&&是一个右值引用。


<!-- more -->

* ParamType既不是指针也不是引用

```cpp
template <typename T>
void f(T param);   // ParamType是一个非指针引用

int x = 27; //int
const int cx = x; // const int
const int& rx = cx; // rx是指向const int的引用

f(x)    // T与Param均为int
f(cx)   // T与Param均为int
f(rx)   // T与Param均为int
```

此时通过传值方式处理, 无论**引用与否, 推到成传值形式** , 且抛弃`const`修饰

* 注意在传参时, 形参是左值引用不能接收右值, 形参是右值引用时不能接受左值
* 但是`&&`被认为是通用引用时可以接受左值右值
* 对于内置变量, `int`, `char`, `double`等, 传指针或引用效率不见得比传值高, 因此形参用`(int a)`即可, 但对于一般的class, 如`string`, 传指针或者引用效率比传值很有优势, 这时候一般对左值用`string&`, 右值用`string&&`, 一般不用`string`。

#### 数组实参

一般的,C语言中有数组退化成指针的规则。
```cpp
const char name[] = "J. P. Briggs";
const char* ptrToName = name;   //数组退化成指针

void myFunc(int param[]);
// 等价于
void myFunc(int* param);
```
模板类型推导中指针不同于数组, 

```cpp
const char name[] = "J. P. Briggs";
void f(T param);
f(name);    // 传指针, T被推导成const char*

void f(T& param);
f(name) // 传数组, T被推导为const char[13], paramType为 const char(&)[13]
```

#### 函数实参

```cpp
void someFunc(int, double); // someFunc是一个函数, 类型是void(int, double)
void f(T param);
f(name);    // 传函数指针, T被推导成void(*)(int double)

void f(T& param);
f(name) // 传函数引用, T被推导为void(&) (int double)
```

### 条款2 理解auto类型推导

auto类型推导和模板类型推导有一个直接的映射关系, 它们直接有非常系统化的转换流程来转换彼此。

注意直接用auto相当于模板推导参数无引用, 对引用会剥去。而`auto&`, `auto&&`分别对应模板推导的剩余两种情形。

在item1 中模板类型推导用下面的函数模板解释
```cpp
template <typename T>
void f(ParmaType param);
```
`ParamType`的三种情况表示`T`, `T&`, `T&&`。

当一个变量使用`auto`声明时, `auto`扮演了模板`T`的角色, 变量的类型则扮演了`ParamType`的角色

```cpp
auto x = 27;
// 转换成
template <typename T>
void func_for_x(T param);
func_for_x(27);

const auto cx = x;
// 转换成
template <typename T>
void func_for_x(const T param);
func_for_x(x);

const auto& rx = x;
// 转换成
template <typename T>
void func_for_x(const T& param);
func_for_x(x);

const int theAnswer = 42;
auto x = theAnswer; // 推导成int
auto y = &theAnswer;    // 推导出const int*
```

即
* 类型说明符是一个指针或引用但不是通用引用 auto&, 这样auto是一个非引用, 但auto& 是一个引用类型。
* 类型说明符是一个通用引用 auto&&
* 类型说明符不是指针或者引用 auto, const auto

* auto之后跟左值, 用auto&, 跟右值用auto&&, auto可以跟左右值但可能涉及到拷贝函数。

同理
```cpp
const char name[] = "R. N. Briggs";

auto arr1 = name;   // arr1类型 const char*
auto& arr2 = name;  // arr2类型const char(&)[13]

void someFunc(int, double);

auto func1 = someFunc;  // func1类型void(*) (int, double)
auto func2 = someFunc;  // func2类型void(&) (int, double)
```

auto类型不同于模板类型推导的情况是, 当用auto声明的变量用花括号初始化, auto类型推导会推导出auto类型为`std::initializer_list`, 但模板不会这样处理。

```cpp
auto x1 = {1,2,3.0};    // 不能工作, 类型必须一致
auto x2 = {11,23,9};    // x2类型为std::initializer_list<int>

template <typename T>
void f(T param);

f({11,23,9});   // 错误, 不能推导出T

template <typename T>
void f(std::initalizer_list<T> initList);

f({11,23,9});   // T被推导成int, 可以运行
```


### 条款2 理解decltype

decltype作用, 给它一个名字或者表达式decltype就会告诉名字或表达式的类型。cpp11中, **decltype最主要用途是用于函数模板返回类型 而这个返回类型依赖于形参**。

显然deltype的实现也是基于模板, 条款2说明直接使用auto可能会剥去引用, 而decltype只是简单的返回表达式的类型, 不会进行模板推导的修改。

```cpp
template <typename Container, typename Index>
auto authAndAccess(Container& c, Index i)
    -> decltype(c[i])       // 推导成int, 而不是int&
{
    authenticateUser();
    return c[i];
}

std::queue<int> d;
authAndAccess(d,5) = 10;    // authAndAccess返回右值, 10赋给右值无法通过编译
```

函数名称前的auto不会做任何类型推导工作, 只是暗示用cpp11的尾置返回类型语法。

cpp14的decltype(auto), 通过一直特殊的规则推导, 不会像模板推导那么死板
```cpp
Widget w;
const Widget& cw = w;

auto myWidget1 = cw;    // auto类型推导, myWidget1类型为Widget

decltype(auto) myWidget2 = cw;  // decltype类型推导, myWidget2类型const Widget&
```

### 条款5 优先考虑auto而非显示类型声明

* auto变量必须初始化, 这使得在编译器检查出未初始化的问题。

### 条款7 区别使用()和{}创建对象

* {} 基于构造函数或者拷贝构造函数的初始化。

```cpp
std::vector<int> v{1,3,5};

class Widget {
private:
    int x{0};
    int y = 0;
    int z(0);   // 错误
};

// 不可拷贝的对象不能用=初始化
std::atomic<int> a1(0); // 对, 使用构造函数初始化
std::atomic<int> a2 = 0;    // 错误
```

* {}可用于一般的初始化, {}的应用更加灵活, 不用区分赋值和拷贝。

```cpp
vector<int> nums = {1,2,3};
```

* `{}`倾向于与`std::initializer_list<>()`进行匹配, `initializer_list`是一种标准库类型，用于表示某种特定类型的值的数组。例如`std::vector`有一个非`initializer_list`构造函数允许你去指定容器的初始大小。也有一个`initializer_list`构造函数允许你使用花括号里面的值初始化容器。

```cpp
std::vector<int> v1(10, 20);    //使用非std::initializer_list构造函数
                                //创建一个包含10个元素的std::vector，
                                //所有的元素的值都是20
std::vector<int> v2{10, 20};    //使用std::initializer_list构造函数
                                //创建包含两个元素的std::vector，
                                //元素的值为10和20

class Widget { 
public:  
    Widget(int i, bool b);                              
    Widget(int i, double d);                            
    Widget(std::initializer_list<long double> il);     
    operator float() const;                             
};

Widget w5(w4);                  //使用小括号，调用拷贝构造函数

Widget w6{w4};                  //使用花括号，调用std::initializer_list构造, (w4转换为float，float转换为double)

Widget w7(std::move(w4));       //使用小括号，调用移动构造函数

Widget w8{std::move(w4)};       //使用花括号，调用std::initializer_list构造
```

`{}`匹配时不允许变窄转换
```cpp
class Widget { 
public: 
    Widget(int i, bool b);                      //同之前一样
    Widget(int i, double d);                    //同之前一样
    Widget(std::initializer_list<bool> il);     //现在元素类型为bool
    …                                           //没有隐式转换函数
};

Widget w{10, 5.0};              //错误！要求变窄转换
```

### 条款8 优先考虑nullptr而非0和NULL

在C语言中,0既可表示为整型, 也可表示为NULL。这可能出现问题。比较典型的是参数匹配, 0是匹配整型还是指针型。

```cpp
void f(int);
void f(bool);
void f(void*);

f(0);   // 会调用f(int)而不是f(void*)
f(NULL);    // 可能不会被编译，一般来说优先调f(int)

f(nullptr); // 调用f(void*)版本
```

### 条款9 优先考虑别名alias而不是typedef 

```cpp
typedef std::unique_ptr<std::unordered_map<std::string, std::string>> UPtrMaps;

using UPtrMaps = std::unique_ptr<std::unordered_map<std::string, std::string>>;

别名可以简化模板写法, 当使用一个类型, 这个类型依赖于模板参数T
```cpp
// typedef 
template <typename T>
struct MyAllocList {
    typedef std::list<T, MyAlloc<T>> type;
};

template <typename T>
class Widget {
private:
// 使用类型typedef std::list<T, MyAlloc<T>> type
    typename MyAllocList<T>::type list;
};

// 别名写法
template <typename T>
using MyAllocList = std::list<T, MyAlloc<T>>;

class Widget {
private:
// 使用类型typedef std::list<T, MyAlloc<T>> type
    MyAllocList<T> list;
};
```

### 条款10 优先考虑限域枚举而非未限域枚举

* 限域枚举中的变量域位于枚举类中(相当于类成员变量), 非限域枚举定义的变量域位于枚举外。

```cpp
// 不限域枚举
enum Color {black, white, red};
auto white = false; // 错误, white 位于枚举外, 已在该域中存在, 重复定义

// 限域枚举
enum class Color { black, white, red};
auto white = false; // 没问题，这个域没有white

Color c = Color::red;

if (c < 14) // 错误, 不能比较, 不接受隐式类型转换
if (static_cast<int>(c) < 14>)  // 有效
```

### 条款11 优先考虑delete函数而非私有声明

使类不可拷贝, 在c++98中声明拷贝构造和赋值操作符为private
```cpp
class basic_ios : public ios_base {
public:
    ...
private:
    basic_ios(const basic_ios& );
    basic_ios& operator=(const basic_ios&);
};
```
cpp11中直接标记为delete函数, 声明为delete相比于定义为private, 在成员函数或者友元函数中也不能访问。
```cpp
class basic_ios : public ios_base {
public:
    ...
private:
    basic_ios(const basic_ios& ) = delete;
    basic_ios& operator=(const basic_ios&) = delete;
};
```

### 条款12 使用override声明重载函数

cpp11提供了overide将派生类函数指定为基类重写版本。编译器甚至会强制转型使基类和子类声明override的函数表现得一致。

```cpp
class Base {
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    void mf4() const;
};
class Derived : public Base {
public:
    virtual void mf1() override;
    virtual void mf2(unsigned int x) override;
    virtual void mf3() && override;
    virtual void mf4() const override;
};
```
通过声明override, 方便重写,编译器会将以上代码转换为
```cpp
class Base {
public:
    virtual void mf1() const;
    virtual void mf2(int x);
    virtual void mf3() &;
    virtual void mf4() const;
};
class Derived : public Base {
public:
    virtual void mf1() const override;
    virtual void mf2(int int x) override;
    virtual void mf3() & override;
    void mf4() const override;
};
```
### 条款13 优先考虑const_iterator而非iterator

const_iterator等价于指向常量的指针, 它们都指向不能被修改的值。

```cpp
template <typename C, typename V>
void findAndInsert(C& container, const V& targetVal, const V& insertVal) {
    using std::cbegin;
    using std::cend;
    auto it = std::find(cbegin(container), cend(container), targetVal);
    container.insert(it, insertVal);
}
```

### 条款14 函数不抛异常用noexcpet

在一个noexcept函数中, 当异常传播到函数外, 优化器不需要保证运行时栈的可展开状态。编译器可以对noexcept函数极致优化。

`swap`函数使noexcept函数的绝佳用地, swap函数往往被认为不抛出异常。

* 在移动构造函数中抛出异常是很危险的，因为会导致空悬的指针，因此最好给移动构造函数添加一个noexcept关键字，如果移动构造函数抛出异常则直接调用teminate函数终止程序，而不是造成指针空悬的状态。

### 条款15 尽可能使用constexpr

constexpr表明一个值不仅仅使常量, 还是编译期可知的。因此, const包含constexpr。

```cpp
int sz; // sz不是const
constexpr auto arraysize = sz;  // 错误, sz编译期不可知

constexpr auto arraysize2 = 10; // 正确, 10编译期可知
```

### 条款16 保证const函数的线程安全

const意味着只读,
* 只有非静态成员函数后面加const（加到非成员函数或静态成员后面会产生编译错误）
* 表示成员函数隐含传入的this指针为const指针，决定了在该成员函数中，
    任意修改它所在的类的成员的操作都是不允许的（因为隐含了对this指针的const引用）；
* 加了const的成员函数可以被非const对象和const对象调用;
    但不加const的成员函数只能被非const对象调用

const成员函数不能修改类成员变量(mutable成员变量除外), 只能被const对象引用, 这隐含了线程安全性。

当存在mutable成员变量, const函数可能不安全时, 应该通过加锁或原子类确保const函数的线程安全。总之const函数应该是线程安全的, 支持并发执行的。

### 条款17 特殊函数的生成

cpp11对特殊成员函数处理的规则如下
* 默认构造函数, 和cpp98一样, 仅当类不存在用户声明的构造函数时自动生成。
* 析构函数: 析构函数默认nonexcept, 类不存在用户声明的析构函数时自动生成, 基类析构为虚函数该类析构也为虚函数。

虚析构函数定义为虚析构函数是为了当用一个基类的指针删除一个派生类的对象时，派生类的析构函数和基类析构函数会被同时调用。 若去掉析构的virtual声明, 则智慧调用基类析构函数。

* 拷贝构造函数 逐成员拷贝非static数据。当用户定义拷贝构造或析构不会自动生成, 类声明移动操作它就是delete
* 拷贝赋值运算符 当用户定义拷贝构造或析构不会自动生成, 类声明移动操作它就是delete

* 移动构造函数和移动赋值运算符 对非static数据逐成员移动, 仅当类没有用户定义的拷贝操作, 移动操作和析构时才自动生成。

综上, 用户声明构造函数会抑制拷贝, 移动的生成。 声明移动, 自定义拷贝会抑制拷贝的自动生成, 声明拷贝, 自定义移动会抑制移动的自动生成。

Rule of three规则, 若声明了拷贝构造函数, 拷贝赋值运算符, 析构函数三者之一, 也应该声明其余两个。

声明移动会抑制拷贝的自动生成, 若需要支持拷贝, 在拷贝操作加上`=default`

```cpp
class Base {
public:
    virtual ~Base() = default;
    Base(Base&&) = default;
    Base& operator=(Base&& ) = default;

    Base(const Base&) = default;
    Base& operator=(const Base&) = default;
}
```
