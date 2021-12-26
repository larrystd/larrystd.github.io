---
title: cpp对象模型:Data和function
date: 2021-01-18
updated: 2021-01-18
tags: 
  - cpp
categories:
  - cpp对象模型
aplayer: true
---

> C++中的Data语义

### Data Member

Data members的布局

```cpp
    class Point3d { 
    public: 
        // ... 
    private: 
        float x; 
        static List<Point3d*> *freeList; 
        float y; 
        static const int chunksize = 250; 
        float z; 
    };   
     
    template <class class_type, class data_type1, class data_type2> 
    char* access_order(data_type1 class_type::*mem1, data_type2 class_type::*mem2 ) 
    { 
        assert (mem1 != mem2); 
        return mem1 < mem2 ? "member 1 occurs first" : "member 2 occurs first"; 
    } 
    // class_type会被绑定为Point3d, data_type1 data_type2会被绑定为 float 
    access_order( &Point3d::z, &Point3d::y); 
```
<!-- more -->

以上，Nonstatic data members在class object中的排列顺序和其被声明的顺序一样，即x,y,z。static data
members则存放在程序的data segment中，和class objects无关。

编译器还可能会合成内部使用的data members，例如vptr。

### Static data members

Static member只有一个实体，并不在class object之中，存取之不需要通过class object。

如果两个class，每个都声明了相同名称static member。当它们都放在data
segment中会导致名称冲突。编译器的解决办法是对每一个static data member进行编码。

Nonstatic data member直接存放在每一个class object中，一般的，存取nonstatic data
member时。编译器需要把class object的起始地址加上data member的偏移量。

### 继承与data member

一般而言，具体继承(concrete inheritance)不会增加空间和时间的额外负担，虚拟继承(virtual
inheritance)则会。但设计继承时，应 **尽量避免重复设计相同操作的函数** ，并且选取合适的函数作为inline函数。

当涉及到多态时，如下所示
```cpp
    class Point2d { 
    public : 
        Point2d( float x = 0.0, float y = 0.0 ) : _x(x), _y(y) { }; 
        // 加上z的保留空间 
        float x() { return _x;} 
        float y() { return _y;} 
        void x( float newX ) { _x = newX } 
        void y( float newY ) { _y = newY } 
     
        virtual float z() { return 0.0; } 
        virtual void z ( float ) { } 
     
        virtual void operator+=( const Point2d& rhs ) { 
            _x += rhs.x(); 
            _y += rhs.y(); 
        } 
    } 
     
    class Point3d : public Point2d { 
    public: 
        Point3d( float x = 0.0, float y = 0.0, float z = 0.0 ) 
            : Point2d(x,y), _z(z) {}; 
        float z() { return _z }; 
        float z( float newZ ) {_z = newZ; } 
        void operator+=( const Point3d& rhs ) { 
            Point2d::operator += ( rhs ); 
            _z += rhs.z(); 
        } 
    protected: 
        float _z; 
    } 
     
    void foo( Point2d& p1, Point2d& p2) 
    { 
        p1 += p2;  // 显然p1,p2可以为2d坐标点，也可以是3d坐标点 
    } 
```
使用virtual带来的额外负担

1. 导入一个和Point2d有关的virtual table，用来存放所声明的virtual function的地址。还要加上几个slot支持runtime type identification 

2. 每个class object导入一个vptr提供执行期的链接

3. 加强constructor，能够为vptr赋初值

4. 加强destructor，能够对vptr,vtbl进行析构。

vptr一般放在class object的起头处或者结尾处。

### 虚拟继承

从不同途径继承来的同一基类，会在子类中存在多份拷贝。这将存在两个问题：其一，浪费存储空间；第二，可能存在一个基类的多份拷贝，二义性。

虚继承底层实现原理与编译器相关，一般通过虚基类指针和虚基类表实现，每个虚继承的子类都有一个虚基类指针（占用一个指针的存储空间，4字节）和虚基类表（不占用类对象的存储空间）。

vbptr指的是虚基类表指针（virtual base table pointer），该指针指向了一个虚基类表（virtual
table），虚表中记录了虚基类与本类的偏移地址；通过偏移地址，这样就找到了虚基类成员。 **派生类不必存储基类的拷贝，转而以地址代替。**
```cpp
    class Point2d { 
    public: ... 
    protected: 
        float _x, _y; 
    }; 
     
    class Vertex : public virtual Point2d { 
    public: ... 
    protected:  
        Vertex *next; 
    }; 
     
    class Point3d : public virtual Point2d { 
    public: ... 
    protected:  
        float _z; 
    }; 
     
    class Vertex3d : public Vertex, public Point3d 
    { 
    public: ... 
    protected:  
        float mumble; 
    }; 
```
![](https://pic1.zhimg.com/v2-2c7679b2673a6a74dbecff8844f90370_b.jpg)

注意到 virtual继承声明在Pointed与Vertex,
Point3d之间，Vertex3d与Vertex和Point3d之间不用声明。后者可以通过正常继承得到virtual继承表和指针。

![](https://pic4.zhimg.com/v2-a9735a633c8684ded44314c7981e29bf_b.jpg)
  
对比虚函数的实现原理：都利用了虚指针（均占用类的存储空间）和虚表（均不占用类的存储空间）。

虚基类依旧存在继承类中，只占用存储空间；虚函数不占用存储空间，虚指针占。

虚基类表存储的是虚基类相对直接继承类的偏移；而虚函数表存储的是虚函数地址。

## Function语意学

C++支持三种类型的member functions: static, non-static, virtual

### Nonstatic Member Functions

C++的设计准则之一就是： nonstatic member function至少和一般的nonmember
function有相同的效率。实际上member function会被编译器转化为nonmember形式处理。
```cpp   
    // 以下来两个函数应该有同等效率
    float magnitude3d( const Point3d *_this) { ... }
    float Point3d::magnitude3d() const { ... }
    
    // 以下是member function被内化为nonmember形式的转化步骤
    // 1. 安插一个额外的参数到member fucntion中，使class object可以调用该函数。
    Point3d
    Point3d::magnitude( Point3d *const this )
    // 若member function为const则变成如下，这也说明const修饰后函数不能改变member data
    // Point3d *const this 指针指向地址不能变，但指向的对象Point3d可以变。
    // const Point3d *const this 指针指向的地址，指向的对象都不能变。
    Point3d
    Point3d::magnitude( const Point3d *const this)
    
    //2. 对每一个nonstatic data member的存取改为由this指针存取
    {
        return sqrt (
            this->_x * this_x);
    }
    
    // 3. 将member function重新写成外部函数，函数名称进行mangling处理。
    extern magnitude_7Point3dFv (
        register Pointe3d *const this);
    // 调用时由 obj.magnitude(); 变为 magnitude_7Point3dFv( &obj );
    // ptr->magnitude(); 变为 magnitude_7Point3dFv( ptr );
```
C++为了支持重载，会对function进行mangling，以提供独一无二的名称。

对nonstatic member function的const修饰实际上是对转化后nonmember function 的obj实参的修饰。

### Static Member Functions

如果Point3::normalize()是一个static member function，以下两个调用操作
```cpp
    obj.normalize(); 
    ptr->normalize(); 
    // 转换为一般的nonmember函数调用 
    normalize_7Point3dSFv(); 
```
Static member functions的主要特性就算它没有this指针，以下的次要特性统统源自其主要特性

  1. 不能直接存取class 中的nonstatic members 
  2. 不能声明为const, volatile或virtual ，因为以上声明本质是对this指针进行修饰 
  3. 不需要经由class object调用 

其地址类型并不是“指向class member function的指针"，而是一个 **non member 函数指针，**
```cpp
     &Point3d::object_count(); 
    // 会得到一个数值，类型为 unsigned int (*) (); 
    // 而不是 unsigned int ( Point3d::*) (); 
```
Static member function由于缺乏this指针，因此差不多等同于nonmember function，成为一个callback函数。

### Virtual Member Functions

如果normaliza()是一个virtual member function，那么调用 ptr->normalize(); 会转化为 (* ptr ->
vptr[ 1 ]) ( ptr ); 第二个ptr表示this指针。

如果magnitude()也是一个virtual function，则
```cpp
    register float mag = magnitude();  // 转化如下,this指针作为形参 
    register float mag = ( *this->vptr[ 2 ] )(this); 
```
执行期多态，以ptr->z()为例，调用操作需要ptr在执行期的某些相关信息。一种直截了当的办法时把需要的信息加到指针ptr身上，但显然这种方法成本太高。另一种方法是把执行期加到对象本身，也就算通过虚函数和表的方法。

在C++中，多态(polymorphism)表示，以一个public base class的指针或reference，寻址出一个derived class
object。没有使用virtual function称消极多态，反之成为积极多态。

一个class只会有一个virtual table，每一个table内含其对应的class object中所有active virtual
functions函数实体的地址。这些active virtual function包括

  1. Class 所定义的函数实体，它会改写可能存在的base class virtual function函数实体。 
  2. 继承自base class的函数实体，即derived class没有改写这些function 
  3. 一个pure_virtual_called()函数实体，可以包括pure virtual function或者异常处理，一旦执行pure_virtual_called()，会停止并退出。 

例
```cpp
    class Point { 
    public: 
        virtual ~Point(); 
        virtual Point& mult( float ) = 0; 
     
        float x() const { return _x; } 
        virtual float y() const { return 0; } 
        virtual float z() const { return 0; } 
    protected: 
        Print( float x = 0.0 ); 
        float _x; 
    }; 
```
其中 virtual destructor赋值slot1,
mult()由于是纯虚函数，被pure_virtual_called()代替放在slot2。x()不是virtual function，不会放在slot中

![](https://pic4.zhimg.com/v2-7b5e7de84688b146332d80c7cbb5db77_b.jpg)
```cpp    
    class Point2d : public Point { 
    public: 
        Point2d( float x = 0.0, float y = 0.0 ) 
            : Point( x ), _y(y) { } 
        ~Point2d(); 
     
        //改写base class virtual functions 
        Point2d& mult( float ); 
        float y() const { return _y; } 
    protected: 
        float _y; 
    } 
```
![](https://pic3.zhimg.com/v2-525e254e78364fc1319d3583296636c2_b.jpg)

在base class Point基础上，Point2d修改了~Point2d(), mult(), y()并拷贝到derive class的virtual
table对应的位置上。
```cpp
    class Point3d : public Point2d { 
    public: 
        Point3d( float x = 0.0, float y = 0.0, float z = 0.0 ) 
            : Point2d( x, y ), _z(z) { } 
        ~Point3d(); 
     
        //改写base class virtual functions 
        Point3d& mult( float ); 
        float z() const { return _z; } 
    protected: 
        float _z; 
    } 
``` 

![](https://pic1.zhimg.com/v2-aa34702f88abdc2df6dfb9d3277ae544_b.jpg)

在继承下，如果有ptr->z()。我们不知道ptr所指对象的真正类型，但是：

1. 可以经由ptr获取virtual table 

2. 每个z()函数都会放在slot4

以上信息， **编译器可以将该调用转化为** ( *ptr-> vptr[4] )(ptr)。至于ptr具体类型执行期判断即可，但无论如何( *ptr->
vptr[4] )(ptr)总会得到想要的ptr->z()。

### Inline 函数

inline函数执行成本比一般函数调用及返回机制带来的负荷低。处理一个inline函数有两个阶段

  1. 分析函数定义，决定函数本质特征。若函数被判断不能成为inline，会被转换成一个static函数。 
  2. inline函数扩展操作，会带来参数求值以及临时对象的管理。 

inline在执行时，每个形式参数会被实际参数取代，例：
```cpp
    
    inline int 
    min( int i , int j ) 
    { 
        return i < j? i : j; 
    } 
     
    //调用操作 
    inline int 
    bar() 
    { 
        int minval; 
        int val1 = 1024; 
        int val2 = 2048; 
        minval = min (val1, val2); //直接代换 minval = val1 < val2 ? val1: val2; 
        minval = min (1024,2048); // 代换之后得到minval = 1024; 
        minval = min( foo(), bar()+1);   
        // 需要导入临时对象，minval = ( t1=foo()),(t2=bar()+1), t1 < t2? t1:t2 
    } 
```
注意到inline函数中如果存在局部变量，可能导致大量临时对象的产生。编写Inline函数时应该尽量写表达式形式，尽量少的使用内部对象，局部变量。

