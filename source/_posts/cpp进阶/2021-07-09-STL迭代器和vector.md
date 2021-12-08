---
title: C++标准库:STL迭代器和vector, list, deque
date: 2021-07-09
updated: 2021-07-09
tags: 
  - stl
categories:
  - cpp advanced
aplayer: true
---

### 迭代器

#### 链表迭代器示例

```cpp
template <class Item>
struct ListIter {
    Item* ptr;  // ptr可以操作容器对象，即链表
    ListIter (Item* p = 0)
        : ptr(p) { }
    
    Item& operator*() const {
        return *ptr;
    }
    Item* operator->() const {
        return ptr;
    }

    ListIter& operator++() {
        ptr = ptr->next();
        return *this;
    }

    bool operator==(const ListIter& i) const {
        return ptr == i.ptr;
    }
}
```

<!-- more -->

#### 迭代器相应型别和traits

* 算法中运行迭代器时，常常需要其相应型别。换句话说, 我们需要**传入一个迭代器就知道它指向对象的相关信息**。对于一般的迭代器对象, 可以在内部封装型别信息(迭代器需要指向对象类型作为模板), 也就是`typedef Category iterator_category;`。但对于指针T*, 我们想知道它的型别是困难的(运行期你基于一个指针无法知道他是int*, 还是char*), 但基于模板编程知道指针`T*`指向T类型信息是可行的, 因为实例化在编译期。换句话说, 运行期我们难以根据指针知道其指向类型, 但编译期可以(基于T*知道T的类型)

这是function template的参数推导机制(argument deducation),或者称萃取traits。萃取是编译期处理方式, 依靠编译器, 可以在编译期间知道指针的类型, 编译出正确的可执行程序。这个过程称之为萃取trait

* 迭代器是算法和容器的桥梁, 算法只需要迭代器来控制当前的对象。而萃取机制则支持, **算法能通过迭代器,包括迭代器指针, 知道当前指向的对象类型等信息(显然指针没有这个能力)**。(指针本身是地址范围, 例如char*表示地址范围是1, 所以指针也能体现一定信息)。在迭代器影响下, 不管容易是什么, 算法都能当线性表处理(也就是begin()和end()指针, begin()++这些)

```cpp
/// 迭代器类型, 传入Category, T作模板
template <class Category, class T, class Distance = ptrdiff_t, class Pointer = T*, class Reference = T&>
struct iterator {
    // typdef 类型
    typedef Category iterator_category;
    typedef T value_type;
    typedef Distance pointer;
    typedef Pointer pointer;
    typedef Reference reference;
};

// 迭代器traits类iterator_traits
/// 传入Iterator类作为模板, 得到Iterator内部含有的容器信息
template<class Iterator>
struct iterator_traits {
    typedef typename Iterator::iterator_category iterator_category;
    typedef typename Iterator::value_type value_type;
    typedef typename Iterator::difference_type difference_type;
    typedef typename Iterator::pointer pointer;
    typedef typename Iterator::reference reference;
};

// 也是struct iterator_traits,原生指针traits偏特化版.当传入T*, 照样可以通过typedef T value_type;得到T的类型
// 编译期行为
template<class T>
struct iterator_traits<T*> {
    typedef typename random_access_iterator_tag iterator_category;
    typedef T value_type;
    typedef ptrdiff_t difference_type;
    typedef T* pointer;
    typedef T& reference;
};
```

gcc4.8中迭代器萃取代码位于`stl_iterator_base_types.h`中
```cpp
  ///  Marking input iterators.
  struct input_iterator_tag { };

  ///  Marking output iterators.
  struct output_iterator_tag { };

  /// Forward iterators support a superset of input iterator operations.
  struct forward_iterator_tag : public input_iterator_tag { };

  /// Bidirectional iterators support a superset of forward iterator
  /// operations.
  struct bidirectional_iterator_tag : public forward_iterator_tag { };

  /// Random-access iterators support a superset of bidirectional
  /// iterator operations.
  struct random_access_iterator_tag : public bidirectional_iterator_tag { };

  template<typename _Category, typename _Tp, typename _Distance = ptrdiff_t,
           typename _Pointer = _Tp*, typename _Reference = _Tp&>    // 迭代器需要Category, Tp等类型信息输入,并维护在对象中
    struct iterator
    {
      /// One of the @link iterator_tags tag types@endlink.
      typedef _Category  iterator_category;
      /// The type "pointed to" by the iterator.
      typedef _Tp        value_type;
      /// Distance between iterators is represented as this type.
      typedef _Distance  difference_type;
      /// This type represents a pointer-to-value_type.
      typedef _Pointer   pointer;
      /// This type represents a reference-to-value_type.
      typedef _Reference reference;
    };

      template<typename _Iterator>
    struct __iterator_traits<_Iterator, true>   // 萃取, 可以直接获取迭代器_Iterator对象存储的类型信息
    {
      typedef typename _Iterator::iterator_category iterator_category;
      typedef typename _Iterator::value_type        value_type;
      typedef typename _Iterator::difference_type   difference_type;
      typedef typename _Iterator::pointer           pointer;
      typedef typename _Iterator::reference         reference;
    };

/// Partial specialization for pointer types. 偏特化迭代器萃取, 可以获取Tp*的Tp类型存储的相关信息, 注意这里输入的类型是Tp*, 不是_Iterator, Tp*指针的迭代器型别显然是random_access_iterator_tag
  template<typename _Tp>
    struct iterator_traits<_Tp*>
    {
      typedef random_access_iterator_tag iterator_category;
      typedef _Tp                         value_type;
      typedef ptrdiff_t                   difference_type;
      typedef _Tp*                        pointer;
      typedef _Tp&                        reference;
    };
```

#### 五种迭代器类型

* input_iterator 只读迭代器 可Operator++
* output_iterator 只写迭代器 可Operator++
* forward_iterator  允许读写(如replace,可Operator++
* bidirectional_iterator 可双向移动 可可Operator++， Operator--
* random_access_iterator 涵盖随机读写，如p+n, p-n等

五种迭代器类型 显然满足一定的继承关系。

```cpp
  ///  Marking input iterators.
  struct input_iterator_tag { };

  ///  Marking output iterators.
  struct output_iterator_tag { };

  /// Forward iterators support a superset of input iterator operations.
  struct forward_iterator_tag : public input_iterator_tag { };

  /// Bidirectional iterators support a superset of forward iterator
  /// operations.
  struct bidirectional_iterator_tag : public forward_iterator_tag { };

  /// Random-access iterators support a superset of bidirectional
  /// iterator operations.
  struct random_access_iterator_tag : public bidirectional_iterator_tag { };
```

type类型

* value_type 迭代器所指对象的型别
* difference_type 表示两个迭代器的距离(的类型)，例如连续空间的容器首尾迭代器距离是最大容量。
* reference_type 迭代器所指对象的引用
* pointer_type 迭代器所指对象的地址
* iterator_category 五类迭代器具体类型

distance函数
```cpp
template<typename _RandomAccessIterator, typename _Distance>
inline void
__advance(_RandomAccessIterator& __i, _Distance __n,
            random_access_iterator_tag)
{
    // concept requirements
    __glibcxx_function_requires(_RandomAccessIteratorConcept<
                _RandomAccessIterator>)
    __i += __n;
}

template<typename _ForwardIterator>
inline _ForwardIterator
next(_ForwardIterator __x, typename
    iterator_traits<_ForwardIterator>::difference_type __n = 1)
{
    std::advance(__x, __n);
    return __x;
}

template<typename _InputIterator, typename _Distance>
inline void
__advance(_InputIterator& __i, _Distance __n, input_iterator_tag)
{
    // concept requirements
    __glibcxx_function_requires(_InputIteratorConcept<_InputIterator>)
    _GLIBCXX_DEBUG_ASSERT(__n >= 0);
    while (__n--)
++__i;
}
```

### 序列式容器

序列型容器
* array(built-in) C++内建
* vector
* heap, priority-queue
* list
* deque
* stack 配接器
* queue 配接器

### vector
vector与array的唯一区别是空间运用的灵活性，array是静态空间，一旦配置不能改变，如需修改，必须手动配置新空间，将元素从旧空间搬到新空间，再释放旧空间。vector相比之，只是将以上操作由系统内部自动完成，但复杂度和array一样，只是方便使用而已。

vector是线性连续空间，以两个迭代器start和finish分别指向配置得来的连续空间目前使用的范围，并以迭代器end_of_storage指向整块空间的尾端。

* 注意, 为避免模板实例化带来的问题, 最好用`size_t`来接收容器的`size()`返回。

```cpp
template <class T, class Alloc = alloc>
class vector {
protected:
    iterator start; // 可用空间头
    iterator finish;    // 使用空间尾
    iterator end_of_storage;    // 可用空间尾
};

使用以上三个迭代器，可以实现常用函数。
```cpp
template <class T, class Alloc = alloc>
class vector {
...
public:
    iterator begin()    { return start; }
    iterator end()  { return finish; }
    size_type size() const
    { return size_type(end() - begin()); }
    size_type capacity() const
    { return size_type(end_of_storage - begin()); }

    bool empty() const {
        return begin() == end();
    }
    reference operator[] (size_type n) {
        return *(begin() + n);
    }
    reference front() {
        return *begin();
    }
    reference back() {
        return *(end() - 1);
    }
```
添加元素，动态增加大小，不是在原空间之后加新空间，而是原大小两倍另外配置较大空间，构造新元素，释放原空间。一旦引起空间重新配置，迭代器将全部失效。

```cpp
vector(size_type n, const T& value) {
    fill_initialize(n ,value);
}

void push_back(const T& x) {
    if (finish != end_of_storage) { // 还有备用空间
        construct(finish, x);
        ++ finish;
    }
    else {  // 无备用空间
        insert_aux(end(), x);
    }
}

void pop_back() {
    --finish;
    destroy(finish);
}

iterator erase(iterator first, iterator last) {} // 擦除[first, last)中所有元素

iterator erase(iterator position) {} // 擦除某个位置上的元素

void clear() {
    erase(first(), end());
}

// 从position开始，插入n个元素，初值为x
template <class T, class Alloc>
void vector<T, Alloc>::insert(iterator position, size_type n, const T& x){}
```

gcc4.8中的vector

gcc4.8首先是一个vector_base, vector_base作用主要是维护了三个指针`start`,`finish`,`end_of_storage`; 分配空间

vector_base迭代器pointer类型是`__gnu_cxx::__alloc_traits<_Tp_alloc_type>::pointer`, 这其实就是一个原始指针而不是迭代器, 因此可以认为vector迭代器就是原始指针, 只不过原始指针视为`_RandomAccessIterator`处理
```cpp
// _Vector_base
template<typename _Tp, typename _Alloc>
    struct _Vector_base // vector_base struct
    {
      typedef typename __gnu_cxx::__alloc_traits<_Alloc>::template
        rebind<_Tp>::other _Tp_alloc_type;  // _Tp_alloc_type貌似是一种Tp的包裹类型
      typedef typename __gnu_cxx::__alloc_traits<_Tp_alloc_type>::pointer
       	pointer;  // pointer 是__gnu_cxx::__alloc_traits<_Tp_alloc_type>的成员, 作为分配空间的指针
      struct _Vector_impl   // 包含三个关键指针
      : public _Tp_alloc_type
      {
	pointer _M_start; //Basevector的三个关键指针
	pointer _M_finish;
	pointer _M_end_of_storage;


	void _M_swap_data(_Vector_impl& __x)  // swap Vector_impl就是swap指针
	{
	  std::swap(_M_start, __x._M_start);
	  std::swap(_M_finish, __x._M_finish);
	  std::swap(_M_end_of_storage, __x._M_end_of_storage);
	}
      };
      
    public:
      typedef _Alloc allocator_type;

      _Tp_alloc_type& // 空间分配
      _M_get_Tp_allocator() _GLIBCXX_NOEXCEPT
      { return *static_cast<_Tp_alloc_type*>(&this->_M_impl); }


      _Vector_base(size_t __n)
      : _M_impl()
      { _M_create_storage(__n); } // BaseVector负责开辟空间

      _Vector_base(size_t __n, const allocator_type& __a)
      : _M_impl(__a)
      { _M_create_storage(__n); }

    private:
      void
      _M_create_storage(size_t __n)
      {
	this->_M_impl._M_start = this->_M_allocate(__n);
	this->_M_impl._M_finish = this->_M_impl._M_start;
	this->_M_impl._M_end_of_storage = this->_M_impl._M_start + __n;
      }
    };

// __alloc_traits
template<typename _Alloc>
  struct __alloc_traits
#if __cplusplus >= 201103L
  : std::allocator_traits<_Alloc>
#endif
  {
    typedef _Alloc allocator_type;
#if __cplusplus >= 201103L
    typedef std::allocator_traits<_Alloc>           _Base_type; // std::allocator_traits<_Alloc>为Base_type, Alloc是包含Tp的类型
    typedef typename _Base_type::value_type         value_type;
    typedef typename _Base_type::pointer            pointer;  // 实际是std::allocator_traits<_Alloc>::pointer, Alloc含有Tp信息
    typedef typename _Base_type::const_pointer      const_pointer;
    typedef typename _Base_type::size_type          size_type;
    typedef typename _Base_type::difference_type    difference_type;

...
//allocator_traits<Alloc>
 
template<typename _Alloc>
struct allocator_traits
{
    /// The allocator type
    typedef _Alloc allocator_type;
    /// The allocated type
    typedef typename _Alloc::value_type value_type;

#define _GLIBCXX_ALLOC_TR_NESTED_TYPE(_NTYPE, _ALT) \
private: \
template<typename _Tp> \
static typename _Tp::_NTYPE _S_##_NTYPE##_helper(_Tp*); \
static _ALT _S_##_NTYPE##_helper(...); \
typedef decltype(_S_##_NTYPE##_helper((_Alloc*)0)) __##_NTYPE; \
public:

_GLIBCXX_ALLOC_TR_NESTED_TYPE(pointer, value_type*) // pointer就是模板指针value_type*

    /**
    * @brief   The allocator's pointer type.
    *
    * @c Alloc::pointer if that type exists, otherwise @c value_type*
    */
    typedef __pointer pointer;
```

vector类继承自vector_base
```cpp
 template<typename _Tp, typename _Alloc = std::allocator<_Tp> >
    class vector : protected _Vector_base<_Tp, _Alloc>  // vector继承自Vector_base, 类型参数是Tp和alloc<Tp>
    {
      // Concept requirements.
      typedef typename _Alloc::value_type                _Alloc_value_type;
      __glibcxx_class_requires(_Tp, _SGIAssignableConcept)
      __glibcxx_class_requires2(_Tp, _Alloc_value_type, _SameTypeConcept)
      
      typedef _Vector_base<_Tp, _Alloc>			 _Base;
      typedef typename _Base::_Tp_alloc_type		 _Tp_alloc_type;
      typedef __gnu_cxx::__alloc_traits<_Tp_alloc_type>  _Alloc_traits;

    public:
      typedef _Tp					 value_type;  // vector维护的一些类型参数
      typedef typename _Base::pointer                    pointer;
      typedef typename _Alloc_traits::const_pointer      const_pointer;
      typedef typename _Alloc_traits::reference          reference;
      typedef typename _Alloc_traits::const_reference    const_reference;
      typedef __gnu_cxx::__normal_iterator<pointer, vector> iterator; // 迭代器类型是指针
      typedef __gnu_cxx::__normal_iterator<const_pointer, vector>
      const_iterator;
      typedef std::reverse_iterator<const_iterator>  const_reverse_iterator;
      typedef std::reverse_iterator<iterator>		 reverse_iterator;
      typedef size_t					 size_type;
      typedef ptrdiff_t					 difference_type;
      typedef _Alloc                        		 allocator_type;

    protected:
      using _Base::_M_allocate;
      using _Base::_M_deallocate;
      using _Base::_M_impl;
      using _Base::_M_get_Tp_allocator;

    vector(size_type __n, const value_type& __value,  // 基于n和value构造
	     const allocator_type& __a = allocator_type())
      : _Base(__n, __a) // Base已经开辟了空间
      { _M_fill_initialize(__n, __value); } // 调用_M_fill_initialize进行赋值
    
    vector(const vector& __x)   // 用另一个vector拷贝构造
      : _Base(__x.size(),
        _Alloc_traits::_S_select_on_copy(__x._M_get_Tp_allocator()))
      { this->_M_impl._M_finish =
	  std::__uninitialized_copy_a(__x.begin(), __x.end(),
				      this->_M_impl._M_start,
				      _M_get_Tp_allocator());
      }
```

插入, 主要是`push_back`和`emplace_back`
```cpp
// 当cpp11, push_back实际就是emplace_back
#if __cplusplus >= 201103L
      void
      push_back(value_type&& __x)
      { emplace_back(std::move(__x)); }

      template<typename... _Args>
        void
        emplace_back(_Args&&... __args);

  template<typename _Tp, typename _Alloc>
    template<typename... _Args>
      void
      vector<_Tp, _Alloc>::
      emplace_back(_Args&&... __args)   // cpp11中, 参数用_Args&&... __args表示
      {
	if (this->_M_impl._M_finish != this->_M_impl._M_end_of_storage)
	  {
	    _Alloc_traits::construct(this->_M_impl, this->_M_impl._M_finish,
				     std::forward<_Args>(__args)...);   // cpp11中
	    ++this->_M_impl._M_finish;
	  }
	else
	  _M_emplace_back_aux(std::forward<_Args>(__args)...);
      }
// 非cpp11
    void
    push_back(const value_type& __x)
    {
    if (this->_M_impl._M_finish != this->_M_impl._M_end_of_storage) // 空间足够
    {
        _Alloc_traits::construct(this->_M_impl, this->_M_impl._M_finish,
                                __x);  // 在this->_M_impl._M_finish迭代器处进行插入x
        ++this->_M_impl._M_finish;
    }
    else
#if __cplusplus >= 201103L
    _M_emplace_back_aux(__x); // 空间不够用调_M_emplace_back_aux
#else
    _M_insert_aux(end(), __x);
#endif
    }

// _M_emplace_back_aux位于vector.tcc之中
template<typename _Tp, typename _Alloc>
    template<typename... _Args>
      void
      vector<_Tp, _Alloc>::
      _M_emplace_back_aux(_Args&&... __args)
      {
	const size_type __len =
	  _M_check_len(size_type(1), "vector::_M_emplace_back_aux");    // 容量变为len, 即2*size
	pointer __new_start(this->_M_allocate(__len));  // 开辟空间
	pointer __new_finish(__new_start);
	__try
	  {
	    _Alloc_traits::construct(this->_M_impl, __new_start + size(),
				     std::forward<_Args>(__args)...);   // 为__new_start+size()地址先构造对象__args
	    __new_finish = 0;

	    __new_finish
	      = std::__uninitialized_move_if_noexcept_a
	      (this->_M_impl._M_start, this->_M_impl._M_finish, // 再将原来位置的元素移动到__new_start中
	       __new_start, _M_get_Tp_allocator());

	    ++__new_finish;
	  }
	__catch(...)    // 扩容空间失败
	  {
	    if (!__new_finish)
	      _Alloc_traits::destroy(this->_M_impl, __new_start + size());
	    else
	      std::_Destroy(__new_start, __new_finish, _M_get_Tp_allocator());
	    _M_deallocate(__new_start, __len);
	    __throw_exception_again;    // 扩容失败抛出异常
	  }
	std::_Destroy(this->_M_impl._M_start, this->_M_impl._M_finish,  // 销毁原来的空间
		      _M_get_Tp_allocator());
	_M_deallocate(this->_M_impl._M_start,
		      this->_M_impl._M_end_of_storage
		      - this->_M_impl._M_start);
	this->_M_impl._M_start = __new_start;   // 每次使用迭代器尽量使用vector::iter, 一次性使用, 防止迭代器失效。
	this->_M_impl._M_finish = __new_finish;
	this->_M_impl._M_end_of_storage = __new_start + __len;
      }

// _M_check_len gcc4.8环境下两倍扩容
    size_type
    _M_check_len(size_type __n, const char* __s) const
    {
      if (max_size() - size() < __n)
	__throw_length_error(__N(__s));

      const size_type __len = size() + std::max(size(), __n);   // 扩充为原来的size+max(size, 1)， 即2*size
      return (__len < size() || __len > max_size()) ? max_size() : __len;
    }
```

赋值, assign, 通过迭代器来赋值.`vector.assign(begin(), end())`，     assign也可以初始化的格式赋值`vector.assign(n, 0)`
```cpp
// 迭代器的形式赋值
    void
    assign(_InputIterator __first, _InputIterator __last)
    { _M_assign_dispatch(__first, __last, __false_type()); }

    template<typename _InputIterator>
        void
        _M_assign_dispatch(_InputIterator __first, _InputIterator __last,
			   __false_type)
        {
	  typedef typename std::iterator_traits<_InputIterator>::
	    iterator_category _IterCategory;
	  _M_assign_aux(__first, __last, _IterCategory());
	}

    template<typename _Tp, typename _Alloc>
    template<typename _InputIterator>
      void
      vector<_Tp, _Alloc>::
      _M_assign_aux(_InputIterator __first, _InputIterator __last,
		    std::input_iterator_tag)
      {
	pointer __cur(this->_M_impl._M_start);
	for (; __first != __last && __cur != this->_M_impl._M_finish;
	     ++__cur, ++__first)
	  *__cur = *__first;    // 赋值
	if (__first == __last)
	  _M_erase_at_end(__cur);
	else
	  insert(end(), __first, __last);
      }
    
// 初始化的形式赋值
      template<typename _Integer>
        void
        _M_assign_dispatch(_Integer __n, _Integer __val, __true_type)
        { _M_fill_assign(__n, __val); }
    
      template<typename _Tp, typename _Alloc>

    void
    vector<_Tp, _Alloc>::
    _M_fill_assign(size_t __n, const value_type& __val)
    {
      if (__n > capacity())
	{
	  vector __tmp(__n, __val, _M_get_Tp_allocator());
	  __tmp.swap(*this);
	}
      else if (__n > size())
	{
	  std::fill(begin(), end(), __val);
	  std::__uninitialized_fill_n_a(this->_M_impl._M_finish,
					__n - size(), __val,
					_M_get_Tp_allocator());
	  this->_M_impl._M_finish += __n - size();
	}
      else
        _M_erase_at_end(std::fill_n(this->_M_impl._M_start, __n, __val));
    }
```

resize, 主要是分配空间，初始化，设置start和finish指针

```cpp
    void
    resize(size_type __new_size)
    {
if (__new_size > size())
    _M_default_append(__new_size - size());
else if (__new_size < size())
    _M_erase_at_end(this->_M_impl._M_start + __new_size);
    }

    template<typename _Tp, typename _Alloc>
    void
    vector<_Tp, _Alloc>::
    _M_default_append(size_type __n)
    {
      if (__n != 0)
	{
	  if (size_type(this->_M_impl._M_end_of_storage
			- this->_M_impl._M_finish) >= __n)
	    {
	      std::__uninitialized_default_n_a(this->_M_impl._M_finish,
					       __n, _M_get_Tp_allocator());
	      this->_M_impl._M_finish += __n;   // 设置_M_finish
	    }
	  else  // 空间不够可能需要扩容
	    {
	      const size_type __len =
		_M_check_len(__n, "vector::_M_default_append");
	      const size_type __old_size = this->size();
	      pointer __new_start(this->_M_allocate(__len));
	      pointer __new_finish(__new_start);
	      __try
		{
		  __new_finish
		    = std::__uninitialized_move_if_noexcept_a
		    (this->_M_impl._M_start, this->_M_impl._M_finish,
		     __new_start, _M_get_Tp_allocator());
		  std::__uninitialized_default_n_a(__new_finish, __n,
						   _M_get_Tp_allocator());
		  __new_finish += __n;
		}
	      __catch(...)
		{
		  std::_Destroy(__new_start, __new_finish,
				_M_get_Tp_allocator());
		  _M_deallocate(__new_start, __len);
		  __throw_exception_again;
		}
	      std::_Destroy(this->_M_impl._M_start, this->_M_impl._M_finish,
			    _M_get_Tp_allocator());
	      _M_deallocate(this->_M_impl._M_start,
			    this->_M_impl._M_end_of_storage
			    - this->_M_impl._M_start);
	      this->_M_impl._M_start = __new_start;
	      this->_M_impl._M_finish = __new_finish;
	      this->_M_impl._M_end_of_storage = __new_start + __len;
	    }
	}
```

### list

vector的迭代器实际是是裸指针, 迭代器操作转为指针操作, 并没有定义真正的迭代器类。list使用的迭代器才是基于迭代器类的最简单对象。
```cpp
// 迭代器
template<typename _Tp>
    struct _List_iterator     // Iterator<list> 类型的迭代器
    {
      typedef _List_iterator<_Tp>                _Self;
      typedef _List_node<_Tp>                    _Node;

      typedef ptrdiff_t                          difference_type;
      typedef std::bidirectional_iterator_tag    iterator_category;     // 这是五种迭代器的哪种
      typedef _Tp                                value_type;
      typedef _Tp*                               pointer;
      typedef _Tp&                               reference;


      explicit
      _List_iterator(__detail::_List_node_base* __x)
      : _M_node(__x) { }

      // Must downcast from _List_node_base to _List_node to get to _M_data.
      reference
      operator*() const
      { return static_cast<_Node*>(_M_node)->_M_data; }

      pointer
      operator->() const
      { return std::__addressof(static_cast<_Node*>(_M_node)->_M_data); }
    // 实现了list下简单的迭代器操作。
      _Self&
      operator++()
      {
	_M_node = _M_node->_M_next;
	return *this;
      }

      _Self
      operator++(int)
      {
	_Self __tmp = *this;
	_M_node = _M_node->_M_next;
	return __tmp;
      }

      _Self&
      operator--()
      {
	_M_node = _M_node->_M_prev;
	return *this;
      }
```

list是一个双向链表。list的好处使每次插入或删除一个元素，就配置或释放一个元素空间，对空间运用一点不浪费，插入删除都是常数时间。list和vector使最常用的容器。

list的节点双向链表。

```cpp
template <class T>
struct __list_node {
    typedef void* void_pointer;
    void_pointer prev;
    void_pointer next;
    T data;
};
```

list是一个**环状双向链表**,只需要一个指针就可以表示完整链表。让指针指向特定节点，就能实现last迭代器。list提供的迭代器是`Bidirectional Iterator`，且插入操作不会造成迭代器失效，删除操作只有指向被删除元素的迭代器失效。

迭代器的作用是用简单的++, --操作屏蔽容器元素内部的复杂操作。迭代器比容器类更底层，容器类的多种函数通过调用迭代器完成。

链表节点结构体
```cpp
    struct _List_node_base
    {
      _List_node_base* _M_next; // 节点的next, prev
      _List_node_base* _M_prev;

      static void
      swap(_List_node_base& __x, _List_node_base& __y) _GLIBCXX_USE_NOEXCEPT;

      void
      _M_transfer(_List_node_base* const __first,
		  _List_node_base* const __last) _GLIBCXX_USE_NOEXCEPT;

      void
      _M_reverse() _GLIBCXX_USE_NOEXCEPT;

      void
      _M_hook(_List_node_base* const __position) _GLIBCXX_USE_NOEXCEPT;

      void
      _M_unhook() _GLIBCXX_USE_NOEXCEPT;
    };

  _GLIBCXX_END_NAMESPACE_VERSION
  } // namespace detail

_GLIBCXX_BEGIN_NAMESPACE_CONTAINER

  /// An actual node in the %list.
  template<typename _Tp>
    struct _List_node : public __detail::_List_node_base
    {
      ///< User's data.
      _Tp _M_data;

#if __cplusplus >= 201103L
      template<typename... _Args>
        _List_node(_Args&&... __args)
	: __detail::_List_node_base(), _M_data(std::forward<_Args>(__args)...) 
        { }
#endif
    };

  // _M_hook用于插入this节点, this节点插入到position前面
    v0oid
    _List_node_base::
    _M_hook(_List_node_base* const __position) _GLIBCXX_USE_NOEXCEPT
    {
      this->_M_next = __position;
      this->_M_prev = __position->_M_prev;
      __position->_M_prev->_M_next = this;
      __position->_M_prev = this;
    }
```

迭代器实现
```cpp
  template<typename _Tp>
  template<typename _Tp>
    struct _List_iterator     // Iterator<list> 类型的迭代器
    {
      typedef _List_iterator<_Tp>                _Self;
      typedef _List_node<_Tp>                    _Node; // 节点类型

      typedef ptrdiff_t                          difference_type;
      typedef std::bidirectional_iterator_tag    iterator_category;     // 这是五种迭代器的哪种
      typedef _Tp                                value_type;
      typedef _Tp*                               pointer;
      typedef _Tp&                               reference; // 迭代器指向的类型

      _List_iterator()
      : _M_node() { }

      explicit
      _List_iterator(__detail::_List_node_base* __x)
      : _M_node(__x) { }

      // Must downcast from _List_node_base to _List_node to get to _M_data.
      reference
      operator*() const
      { return static_cast<_Node*>(_M_node)->_M_data; }

      pointer
      operator->() const
      { return std::__addressof(static_cast<_Node*>(_M_node)->_M_data); }

    // 迭代器一些操作
      _Self&
      operator++()
      {
	_M_node = _M_node->_M_next;
	return *this;
      }

      _Self
      operator++(int)
      {
	_Self __tmp = *this;
	_M_node = _M_node->_M_next;
	return __tmp;
      }
    // 迭代器指向的节点_List_node_base
  __detail::_List_node_base* _M_node;
```

list class
```cpp
  template<typename _Tp, typename _Alloc = std::allocator<_Tp> >
    class list : protected _List_base<_Tp, _Alloc>
    {
      // concept requirements
      typedef typename _Alloc::value_type                _Alloc_value_type;
      __glibcxx_class_requires(_Tp, _SGIAssignableConcept)
      __glibcxx_class_requires2(_Tp, _Alloc_value_type, _SameTypeConcept)

      typedef _List_base<_Tp, _Alloc>                    _Base;
      typedef typename _Base::_Tp_alloc_type		 _Tp_alloc_type;
      typedef typename _Base::_Node_alloc_type		 _Node_alloc_type;

    public:
      typedef _Tp                                        value_type;
      typedef typename _Tp_alloc_type::pointer           pointer;
      typedef typename _Tp_alloc_type::const_pointer     const_pointer;
      typedef typename _Tp_alloc_type::reference         reference;
      typedef typename _Tp_alloc_type::const_reference   const_reference;
      typedef _List_iterator<_Tp>                        iterator;      // 实例化迭代器类型, _List_iterator<_Tp>, 从而可以用到前面的迭代器
      typedef _List_const_iterator<_Tp>                  const_iterator;
      typedef size_t                                     size_type;
      typedef ptrdiff_t                                  difference_type;
      typedef _Alloc                                     allocator_type;

    
  // 初始化
        list(size_type __n, const value_type& __value,
	   const allocator_type& __a = allocator_type())
      : _Base(_Node_alloc_type(__a))
      { _M_fill_initialize(__n, __value); }           // 初始化

  void
      _M_fill_initialize(size_type __n, const value_type& __x)
      {
	for (; __n; --__n)
	  push_back(__x); // 调用push_back
      }

  void
    push_back(const value_type& __x)
    { this->_M_insert(end(), __x); }
  
       template<typename... _Args>
       void
       _M_insert(iterator __position, _Args&&... __args)
       {
	 _Node* __tmp = _M_create_node(std::forward<_Args>(__args)...); // 创建一个节点
	 __tmp->_M_hook(__position._M_node);  // tmp插入节点
       }

      // begin(), 返回的是迭代器对象, 即_M_node._M_next
      iterator
      begin() _GLIBCXX_NOEXCEPT
      { return iterator(this->_M_impl._M_node._M_next); }

      const_reference
      front() const
      { return *begin(); }

    // 以上常用函数, list容器都是通过操作迭代器
      template<typename _Tp, typename _Alloc>
    void
    list<_Tp, _Alloc>::
    resize(size_type __new_size)
    {
      iterator __i = begin();
      size_type __len = 0;
      for (; __i != end() && __len < __new_size; ++__i, ++__len)
        ;
      if (__len == __new_size)
        erase(__i, end());
      else                          // __i == end()
	_M_default_append(__new_size - __len);
    }

    void
  pop_back()
  { this->_M_erase(iterator(this->_M_impl._M_node._M_prev)); }

    void
  push_front(const value_type& __x)
  { this->_M_insert(begin(), __x); }
```

#### deque 

deque是一个双端队列，由一段一段定量连续空间构成，一旦有必要在deque前端或尾端增加新空间，便配置一段定量连续空间，串联在deque的头端或尾端。这避开了vector重新配置，复制，释放的复杂度。deque优势是头和尾插入删除都很容易, 但不能向vector一样方便索引查找。

deque采用一块映射结构，其中每个node都是指针，指向一段连续线性空间，成为缓冲区。缓冲区才是deque的储存空间主体。

deque迭代器维护四个指针
```cpp
T* cur; // 指向缓冲区当前元素
T* first;   // 指向缓冲区的头
T* last;    // 指向缓冲区的尾
map_pointer node;   // 指向映射结构
```
deque首先维护一个指向映射结构的指针，然后维护start,finish两个迭代器，指向第一缓冲区的第一个元素和最后缓冲区的最后一个元素的下一个位置。

```cpp
template <class T, class Alloc = alloc, size_t BufSize = 0>
class deque {
public:
    typedef T value_type;
    typedef value_type* pointer;
    typedef size_t size_type;

public:
    typedef __deque_iterator<T, T&, T*, BufSiz> iterator;
protected:
    typedef pointer* map_pointer;
    iterator start; // 第一个节点
    iterator finsh;
    map_pointer map;
    size_type map_size;

    iterator begin()    {return start;}
    iterator end()  { return finish; }
    ...
}
```

#### stack 和 queue

stack系以底部容器完成工作，修改某物接口以形成另一种风貌称之为adapter配接器。

```cpp
template <class T, class Sequence = deque<T>>
class stack {
protected:
    Sequence c;
public:
    bool empty() const { return c.empty(); }
    reference top() {
        return c.back();
    }
    void push(const value_type& x) {
        c.push_back(x);
    }
    void pop() {
        c.pop_back();
    }
};
```
queue同理，先进先出。
```cpp
template <class T, class Sequence = deque<T>>
class stack {
protected:
    Sequence c;
public:
    bool empty() const { return c.empty(); }
    reference front() {
        return c.front();
    }
    reference back() {
        return c.back();
    }
    void push(const value_type& x) {
        c.push_back(x);
    }
    void pop() {
        c.pop_front();
    }
};
```

#### priority_queue优先队列

优先队列是一个有权值的队列。由于这是一个队列，所以只允许尾端加入元素，顶端取出元素。而内部元素排列并非按照推入的次序排列，而是自动按照权值排列。

优先队列底层用最大堆实现，堆元素的插入其实是上溯+下溯两个操作。先自底向上判断父节点元素是否大于子节点，若否，修改之并以子节点自顶向下修正。

```cpp
template <class T, class Sequence = vector<T>,
class Compare = less<typename Sequence::value_type>>

class priority_queue {
protected:
    Sequence c; // 底层容器
    Compare comp;   // 元素大小比较标准
public:
    priority_queue() : c() {}
    explicit priority_queue(const Compare& x) : c(), comp(x) {}
    template <class InputIterator>
    priority_queue(InputIterator first, InputIterator last)
    : c (first, last) {
        make_heap(c.begin(), c.end(), comp);
    }

    bool empty() const {
        return c.empty();
    }
    size_type size() const {
        return c.size();
    }
    const_reference top() const {
        return c.front();
    }
    void push(const value_type& x) {
        c.push_back(x);
        // 重排heap
        push_heap(c.begin(), c.end(), comp);
    }
    void pop() {
        pop_heap(c.begin, c.end(), comp);
        c.pop_back();
    }
}
```