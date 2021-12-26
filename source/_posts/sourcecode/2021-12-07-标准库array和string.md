---
title: 标准库array, bitset和string, random
date: 2021-12-07
updated: 2021-12-07
tags: 
  - C++标准库
  - cpp
categories:
  - source code
aplayer: true
---

> 基于g++ 4.8, 读源码是为了更好了使用标准库

### array

array和vector, deque, map等容器不同, 其分配内存在栈上, 且编译期配置内存。不需要STL的allocator空间配置器。

由于是编译期确定array的size, 顾通过模板元编程直接在模板中设置大小即可。另外由于编译期确定了大小, array不需要有push_back这种操作, 只需要通过`operator[]`或者`at()`赋值即可。

```cpp
template<typename _Tp, std::size_t _Nm>
    struct array
    {
      typedef _Tp 	    			      value_type;
      typedef value_type*			      pointer;
      typedef const value_type*                       const_pointer;
      typedef value_type&                   	      reference;
      typedef const value_type&             	      const_reference;
      typedef value_type*                             iterator;
      typedef const value_type*                       const_iterator;
      typedef std::size_t                    	      size_type;
      typedef std::ptrdiff_t                   	      difference_type;

      reference
      at(size_type __n) // 返回的是reference
      {
	if (__n >= _Nm)
	  std::__throw_out_of_range(__N("array::at"));
	return _AT_Type::_S_ref(_M_elems, __n);
      }

    pointer
    data() noexcept // 返回数组引用
    { return std::__addressof(_AT_Type::_S_ref(_M_elems, 0)); }

    constexpr size_type 
    size() const noexcept { return _Nm; } // _Nm是模板参数

    template<std::size_t _Int, typename _Tp, std::size_t _Nm> // 通过模板参数实现某个位置元素
    constexpr _Tp&
    get(array<_Tp, _Nm>& __arr) noexcept
    {
      static_assert(_Int < _Nm, "index is out of bounds");
      return _GLIBCXX_STD_C::__array_traits<_Tp, _Nm>::
	_S_ref(__arr._M_elems, _Int); // _Int是模板参数, 这里通过模板参数获取索引元素
    }

    template<typename _Tp, std::size_t _Nm>
    struct __array_traits
    {
      typedef _Tp _Type[_Nm];

      static constexpr _Tp&
      _S_ref(const _Type& __t, std::size_t __n) noexcept  // 获取n位置的元素
      { return const_cast<_Tp&>(__t[__n]); }
    };

// 测试
#include <iostream>
#include <array>
using namespace std;
int main()
{
    std::array<int, 4> values{};  // 可以通过成员初始列初始化
    //初始化 values 容器为 {0,1,2,3}
    for (int i = 0; i < values.size(); i++) {
        values.at(i) = i;
    }
    //使用 get() 模板重载函数输出指定位置元素
    cout << get<3>(values) << endl;
    //如果容器不为空，则输出容器中所有的元素
    if (!values.empty()) {
        for (auto val = values.begin(); val < values.end(); val++) {
            cout << *val << " ";
        }
    }
}
```
<!-- more -->
### bitset

bitset是替代`vector<bool>`的一种办法, 由于C++标准对于`vector<bool>`值有其特殊的实现方法。目的是为了减小空间的耗用。特殊版本内部只使用一个bit来存储一个元素，所以通常要比一般的bool值小8倍之多(基于Bit而不是byte)。但是以上基于bit而不是byte的存储硬是要提供`vector`的接口, 因此其references和iterators经过了特殊的处理，并不是bool值的实际地址，而是一个“代理对象”。这增加了运行效率, 尽管减少了空间消耗, 但我们往往更关注运行效率。

`vector<bool>`的原罪是内存特殊实现硬要实现vector的接口, bitset内部也是基于bit, 但没有复杂的`vector`接口, 不存在所谓"代理对象"的问题。但是bitset和array一样, 都必须在编译期把size作为模板参数输入。

```cpp
template<size_t _Nb>
    class bitset
    : private _Base_bitset<_GLIBCXX_BITSET_WORDS(_Nb)>
    {
    private:
      typedef _Base_bitset<_GLIBCXX_BITSET_WORDS(_Nb)> _Base;
    
    // 某个位置元素, 就是0/1
      reference
      operator[](size_t __position)
      { return reference(*this, __position); }  // 返回reference对象
    
  
  // reference class
     class reference
      {
    friend class bitset;

    _WordT*	_M_wp;
    size_t 	_M_bpos;
      reference(bitset& __b, size_t __pos) _GLIBCXX_NOEXCEPT
    {
      _M_wp = &__b._M_getword(__pos);
      _M_bpos = _Base::_S_whichbit(__pos);
    }
      }

  // 常用函数
        _GLIBCXX_CONSTEXPR size_t
      size() const _GLIBCXX_NOEXCEPT
      { return _Nb; } // _Nb是模板参数

  // 可用string, char* 等来构造bitset
  template<class _CharT, class _Traits, class _Alloc>
	explicit
	bitset(const std::basic_string<_CharT, _Traits, _Alloc>& __s,
	       size_t __position = 0)
	: _Base()
	{
	  if (__position > __s.size())
	    __throw_out_of_range(__N("bitset::bitset initial position "
				     "not valid"));
	  _M_copy_from_string(__s, __position,
			      std::basic_string<_CharT, _Traits, _Alloc>::npos,
			      _CharT('0'), _CharT('1'));
	}

  // bitset 实现了一些位运算
        bitset<_Nb>&
      operator&=(const bitset<_Nb>& __rhs) _GLIBCXX_NOEXCEPT
      {
	this->_M_do_and(__rhs);
	return *this;
      }

      bitset<_Nb>&
      operator|=(const bitset<_Nb>& __rhs) _GLIBCXX_NOEXCEPT
      {
	this->_M_do_or(__rhs);
	return *this;
      }

      bitset<_Nb>&
      operator^=(const bitset<_Nb>& __rhs) _GLIBCXX_NOEXCEPT
      {
	this->_M_do_xor(__rhs);
	return *this;
      }

      bitset<_Nb>&
      operator<<=(size_t __position) _GLIBCXX_NOEXCEPT
      {
	if (__builtin_expect(__position < _Nb, 1))
	  {
	    this->_M_do_left_shift(__position);
	    this->_M_do_sanitize();
	  }
	else
	  this->_M_do_reset();
	return *this;
      }

    // 实现了to_string()函数转为字符表示
  template<class _CharT, class _Traits, class _Alloc>
	std::basic_string<_CharT, _Traits, _Alloc>
	to_string() const
	{
	  std::basic_string<_CharT, _Traits, _Alloc> __result;
	  _M_copy_to_string(__result, _CharT('0'), _CharT('1'));
	  return __result;
	}

```

使用示例
```cpp
#include <iostream>       // std::cout
#include <string>         // std::string
#include <bitset>         // std::bitset

int main ()
{
  std::bitset<16> foo;
  std::bitset<16> bar (0xfa2);
  std::bitset<16> baz (std::string("0101111001"));

  std::cout << "foo: " << foo << '\n';
  std::cout << "bar: " << bar << '\n';
  std::cout << "baz: " << baz << '\n';
  std::cout << bar[8] << '\n';
  std::cout << baz.to_string() << '\n';

  return 0;
}

输出
foo: 0000000000000000
bar: 0000111110100010
baz: 0000000101111001
1
0000000101111001
```

### string

string在标准库中是实例化的basic_string class, 后者是一个模板啊类。
```cpp
  /// A string of @c char
  typedef basic_string<char>    string;   

#ifdef _GLIBCXX_USE_WCHAR_T
  /// A string of @c wchar_t
typedef basic_string<wchar_t> wstring;   
// wchar_t是Unicode字符的数据类型，不再是ascii码,它实际定义在里：
typedef unsigned short wchar_t;  //2^16 = 65536, ASCII码是2^8=128
```

实现
```cpp
template<typename _CharT, typename _Traits, typename _Alloc>
    class basic_string
    {
      typedef typename _Alloc::template rebind<_CharT>::other _CharT_alloc_type;

      // Types:
    public:
      typedef _Traits					    traits_type;
      typedef typename _Traits::char_type		    value_type;
      typedef _Alloc					    allocator_type;
      typedef typename _CharT_alloc_type::size_type	    size_type;
      typedef typename _CharT_alloc_type::difference_type   difference_type;
      typedef typename _CharT_alloc_type::reference	    reference;  // 推导出相当于_CharT&
      typedef typename _CharT_alloc_type::const_reference   const_reference;
      typedef typename _CharT_alloc_type::pointer	    pointer;
      typedef typename _CharT_alloc_type::const_pointer	    const_pointer;

    // @brief 构造函数, 调用construct开辟空间和赋值
    template<typename _CharT, typename _Traits, typename _Alloc>
    basic_string<_CharT, _Traits, _Alloc>::
    basic_string(size_type __n, _CharT __c, const _Alloc& __a)
    : _M_dataplus(_S_construct(__n, __c, __a), __a) // 初始化
    { }

    // contruct
    template<typename _CharT, typename _Traits, typename _Alloc>
    _CharT*
    basic_string<_CharT, _Traits, _Alloc>::
    _S_construct(size_type __n, _CharT __c, const _Alloc& __a)
    {
#if _GLIBCXX_FULLY_DYNAMIC_STRING == 0
      if (__n == 0 && __a == _Alloc())
	return _S_empty_rep()._M_refdata();
#endif
      // Check for out_of_range and length_error exceptions.
      _Rep* __r = _Rep::_S_create(__n, size_type(0), __a);  // 这里会分配空间
      if (__n)
	_M_assign(__r->_M_refdata(), __n, __c); // assign 赋值

      __r->_M_set_length_and_sharable(__n);
      return __r->_M_refdata();
    }


    /**
    *  @brief  Append a single character.
    *  @param __c  Character to append.
    */
      void
      push_back(_CharT __c) // push_back一个字符
      { 
	const size_type __len = 1 + this->size();
	if (__len > this->capacity() || _M_rep()->_M_is_shared())
	  this->reserve(__len); // 分配空间, 动用allocator分配空间
	traits_type::assign(_M_data()[this->size()], __c);  // 赋值
	_M_rep()->_M_set_length_and_sharable(__len);
      }

      // 在最后加入字符串
      basic_string&
      append(initializer_list<_CharT> __l)  // 可以插入成员初值列, {'a','b','c'}这种
      { return this->append(__l.begin(), __l.size()); }
        
      template<typename _CharT, typename _Traits, typename _Alloc>
    basic_string<_CharT, _Traits, _Alloc>&
    basic_string<_CharT, _Traits, _Alloc>::
    append(const basic_string& __str)   // 接入basic_string str
    {
      const size_type __size = __str.size();
      if (__size)
	{
	  const size_type __len = __size + this->size();
	  if (__len > this->capacity() || _M_rep()->_M_is_shared())
	    this->reserve(__len);
	  _M_copy(_M_data() + this->size(), __str._M_data(), __size);
	  _M_rep()->_M_set_length_and_sharable(__len);
	}
      return *this;
    }

    // operator+内部调用的是append()
      template<typename _CharT, typename _Traits, typename _Alloc>
    basic_string<_CharT, _Traits, _Alloc>
    operator+(_CharT __lhs, const basic_string<_CharT, _Traits, _Alloc>& __rhs)
    {
      typedef basic_string<_CharT, _Traits, _Alloc> __string_type;
      typedef typename __string_type::size_type	  __size_type;
      __string_type __str;
      const __size_type __len = __rhs.size();
      __str.reserve(__len + 1);
      __str.append(__size_type(1), __lhs);
      __str.append(__rhs);
      return __str;
    }   

    // 插入字符
      basic_string&
      insert(size_type __pos, size_type __n, _CharT __c)
      { return _M_replace_aux(_M_check(__pos, "basic_string::insert"),
			      size_type(0), __n, __c); }
        
    template<typename _CharT, typename _Traits, typename _Alloc>
    basic_string<_CharT, _Traits, _Alloc>&
    basic_string<_CharT, _Traits, _Alloc>::
    _M_replace_aux(size_type __pos1, size_type __n1, size_type __n2,
		   _CharT __c)
    {
      _M_check_length(__n1, __n2, "basic_string::_M_replace_aux");
      _M_mutate(__pos1, __n1, __n2);  // 重新开辟空间赋值
      if (__n2)
	_M_assign(_M_data() + __pos1, __n2, __c); // assign赋值__c
      return *this;
    }

    template<typename _CharT, typename _Traits, typename _Alloc>
    void
    basic_string<_CharT, _Traits, _Alloc>::
    _M_mutate(size_type __pos, size_type __len1, size_type __len2)
    {
      const size_type __old_size = this->size();
      const size_type __new_size = __old_size + __len2 - __len1;
      const size_type __how_much = __old_size - __pos - __len1;

      if (__new_size > this->capacity() || _M_rep()->_M_is_shared())
	{
	  // Must reallocate.
	  const allocator_type __a = get_allocator();
	  _Rep* __r = _Rep::_S_create(__new_size, this->capacity(), __a);

	  if (__pos)
	    _M_copy(__r->_M_refdata(), _M_data(), __pos);
	  if (__how_much)
	    _M_copy(__r->_M_refdata() + __pos + __len2,
		    _M_data() + __pos + __len1, __how_much);

	  _M_rep()->_M_dispose(__a);
	  _M_data(__r->_M_refdata());
	}
      else if (__how_much && __len1 != __len2)
	{
	  // Work in-place. 原地移动
	  _M_move(_M_data() + __pos + __len2,
		  _M_data() + __pos + __len1, __how_much);
	}
      _M_rep()->_M_set_length_and_sharable(__new_size);
    }

  // find函数
  size_type
      find(const basic_string& __str, size_type __pos = 0) const
	_GLIBCXX_NOEXCEPT
      { return this->find(__str.data(), __pos, __str.size()); }
  
    template<typename _CharT, typename _Traits, typename _Alloc>
    typename basic_string<_CharT, _Traits, _Alloc>::size_type
    basic_string<_CharT, _Traits, _Alloc>::
    find(const _CharT* __s, size_type __pos, size_type __n) const
    {
      __glibcxx_requires_string_len(__s, __n);
      const size_type __size = this->size();
      const _CharT* __data = _M_data();

      if (__n == 0)
	return __pos <= __size ? __pos : npos;

      if (__n <= __size)
	{
	  for (; __pos <= __size - __n; ++__pos)
	    if (traits_type::eq(__data[__pos], __s[0])
		&& traits_type::compare(__data + __pos + 1, 
					__s + 1, __n - 1) == 0) // 这里通过string compare可以实现子字符串的查找, 但是效率较低
	      return __pos;
	}
      return npos;  // npos字符串 class终止符
    }
```

迭代器和其他常用函数
```cpp
      // Iterators:
      /**
       *  Returns a read/write iterator that points to the first character in
       *  the %string.  Unshares the string.
       */
      iterator
      begin() _GLIBCXX_NOEXCEPT
      {
	_M_leak();
	return iterator(_M_data());

  // _M_data()来自分配allocator内存的地址
        struct _Alloc_hider : _Alloc
      {
	_Alloc_hider(_CharT* __dat, const _Alloc& __a)
	: _Alloc(__a), _M_p(__dat) { }

	_CharT* _M_p; // The actual data. String 内部维护的指针_CharT*
      };

    public:
      // Data Members (public):
      // NB: This is an unsigned type, and thus represents the maximum
      // size that the allocator can hold.
      ///  Value returned by various member functions when they fail.
      static const size_type	npos = static_cast<size_type>(-1);

    private:
      // Data Members (private):
      mutable _Alloc_hider	_M_dataplus;

      _CharT*
      _M_data() const
      { return  _M_dataplus._M_p; }

      _CharT*
      _M_data(_CharT* __p)
      { return (_M_dataplus._M_p = __p); }
      }

    // stoi string转为int
    inline int 
  stoi(const wstring& __str, size_t* __idx = 0, int __base = 10)
  { return __gnu_cxx::__stoa<long, int>(&std::wcstol, "stoi", __str.c_str(),
					__idx, __base); }
    inline float
  stof(const wstring& __str, size_t* __idx = 0)
  { return __gnu_cxx::__stoa(&std::wcstof, "stof", __str.c_str(), __idx); }

  inline double
  stod(const wstring& __str, size_t* __idx = 0)
  { return __gnu_cxx::__stoa(&std::wcstod, "stod", __str.c_str(), __idx); }
```

string class的问题是, 实现的不够好, 常见的split等字符串操作没有实现; 对于读频繁和写频繁没有做到兼容, 对于读频繁的操作提供了不少复杂写操作, 对于写频繁又没有提供完全舒适的环境。且扩展性不好, 因此各公司往往根据实际情况自己实现字符串。常见的库几乎都会自己实现字符串, 比如spdlog, folly, fmt等。常见的是Google的StringView, Facebook的FbString。

但是string内部维护的也是char*, 效率低也低不了哪里去。更多是接口实现上的缺陷。以及, 可能太复杂了, string是基础底层类, 根据实际情况重新实现一个也未尝不可。

### 随机数生成 random

一般的使用
```cpp
#include <iostream>
#include <random>
using namespace std;
 
int main( ){
    default_random_engine e; // 设置随机数引擎
    uniform_int_distribution<unsigned> u(0, 9);
    for(int i=0; i<10; ++i)
        cout<<u(e)<<endl;
    return 0;
}
```

部分实现

default_random_engine
```cpp
// 这里的引擎default_random_engine类似于cin, cout
  typedef linear_congruential_engine<uint_fast32_t, 16807UL, 0UL, 2147483647UL> minstd_rand0;
  typedef minstd_rand0 default_random_engine;

  // linear_congruential_engine
    template<typename _UIntType, _UIntType __a, _UIntType __c, _UIntType __m>
    class linear_congruential_engine
    {
      static_assert(std::is_unsigned<_UIntType>::value, "template argument "
		    "substituting _UIntType not an unsigned integral type");
      static_assert(__m == 0u || (__a < __m && __c < __m),
		    "template argument substituting __m out of bounds");

    public:
      /** The type of the generated random value. */
      typedef _UIntType result_type;

      /** The multiplier. */
      static constexpr result_type multiplier   = __a;
      /** An increment. */
      static constexpr result_type increment    = __c;
      /** The modulus. */
      static constexpr result_type modulus      = __m;
      static constexpr result_type default_seed = 1u;

      /**
       * @brief Constructs a %linear_congruential_engine random number
       *        generator engine with seed @p __s.  The default seed value
       *        is 1.
       *
       * @param __s The initial seed value.
       */
      explicit
      linear_congruential_engine(result_type __s = default_seed)
      { seed(__s); }

      /**
       * @brief Constructs a %linear_congruential_engine random number
       *        generator engine seeded from the seed sequence @p __q.
       *
       * @param __q the seed sequence.
       */
      template<typename _Sseq, typename = typename
	std::enable_if<!std::is_same<_Sseq, linear_congruential_engine>::value>
	       ::type>
        explicit
        linear_congruential_engine(_Sseq& __q)
        { seed(__q); }

      /**
       * @brief Reseeds the %linear_congruential_engine random number generator
       *        engine sequence to the seed @p __s.
       *
       * @param __s The new seed.
       */
      void
      seed(result_type __s = default_seed);
  ```

uniform_int_distribution 均匀分布, 内部通过`operator()`形成了仿函数,参数是_UniformRandomNumberGenerator
  ```cpp
    template<typename _IntType = int>
    class uniform_int_distribution
    {
      static_assert(std::is_integral<_IntType>::value,
		    "template argument not an integral type");

    public:
      /** The type of the range of the distribution. */
      typedef _IntType result_type;
      /** Parameter type. */
      struct param_type
      {
	typedef uniform_int_distribution<_IntType> distribution_type;

	explicit
	param_type(_IntType __a = 0,
		   _IntType __b = std::numeric_limits<_IntType>::max())
	: _M_a(__a), _M_b(__b)  // 初始化
	{
	  _GLIBCXX_DEBUG_ASSERT(_M_a <= _M_b);
	}

  // 实现operator()
  template<typename _UniformRandomNumberGenerator>
	result_type
	operator()(_UniformRandomNumberGenerator& __urng)
        { return this->operator()(__urng, _M_param); }

      template<typename _UniformRandomNumberGenerator>
	result_type
	operator()(_UniformRandomNumberGenerator& __urng,
		   const param_type& __p);
```