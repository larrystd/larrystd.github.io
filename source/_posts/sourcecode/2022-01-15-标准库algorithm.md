---
title: 标准库algorithm
date: 2022-01-14
updated: 2022-01-14
tags: 
  - C++标准库
  - cpp
categories:
  - source code
aplayer: true
---

> 基于g++ 4.8, 读源码是为了更好了使用标准库, algorithm是C++标准库的算法库

先看algorithm.h下的std::命名空间, 
```cpp
#ifndef _GLIBCXX_ALGORITHM
#define _GLIBCXX_ALGORITHM 1

#pragma GCC system_header

#include <utility> // UK-300.
#include <bits/stl_algobase.h>
#include <bits/stl_algo.h>

#ifdef _GLIBCXX_PARALLEL
# include <parallel/algorithm>
#endif

#endif /* _GLIBCXX_ALGORITHM */
```

### stl_algobase.h

stl_algobase.h中的算法主要有以下

#### min和max

```cpp

  template<typename _Tp>
    inline const _Tp&
    min(const _Tp& __a, const _Tp& __b)
    {
      // concept requirements
      __glibcxx_function_requires(_LessThanComparableConcept<_Tp>)
      //return __b < __a ? __b : __a;
      if (__b < __a)  
	return __b;
      return __a;
    }

    template<typename _Tp, typename _Compare>
    inline const _Tp&
    min(const _Tp& __a, const _Tp& __b, _Compare __comp)
    {
      //return __comp(__b, __a) ? __b : __a;
      // __comp应该重载为 lhs < rhs
      if (__comp(__b, __a)) // __b < __a
	return __b;
      return __a;
    }

      template<typename _Tp, typename _Compare>
    inline const _Tp&
    max(const _Tp& __a, const _Tp& __b, _Compare __comp)
    // __comp同样应该重载为 lhs < rhs
    {
      //return __comp(__a, __b) ? __b : __a;
      if (__comp(__a, __b)) // means __a < __b
	return __b;
      return __a;
    }
```

<!-- more -->

### stl_algo.h

#### find

```cpp
  template<typename _InputIterator, typename _Tp>
    inline _InputIterator
    __find(_InputIterator __first, _InputIterator __last,
	   const _Tp& __val, input_iterator_tag)
    {
      while (__first != __last && !(*__first == __val))
	++__first;
      return __first;
    }

// __pred是一个返回为true/false的adapter
// find_if就是找到第一个使__pred为true的元素
    template<typename _InputIterator, typename _Predicate>
    inline _InputIterator
    __find_if(_InputIterator __first, _InputIterator __last,
	      _Predicate __pred, input_iterator_tag)
    {
      while (__first != __last && !bool(__pred(*__first)))
	++__first;
      return __first;
    }
```

#### search

search_n, 查找连续的count个val序列, 例如`3334`count=4, val=3; 在迭代器区间`[first,last)`第一个出现的位置。
```cpp
  template<typename _ForwardIterator, typename _Integer, typename _Tp>
    _ForwardIterator
    __search_n(_ForwardIterator __first, _ForwardIterator __last,
	       _Integer __count, const _Tp& __val,
	       std::forward_iterator_tag)
    {
      __first = _GLIBCXX_STD_A::find(__first, __last, __val);
      while (__first != __last)
	{
	  typename iterator_traits<_ForwardIterator>::difference_type
	    __n = __count;  // n为val连续的个数
	  _ForwardIterator __i = __first;
	  ++__i;
    // 需要有n个连续的__val
	  while (__i != __last && __n != 1 && *__i == __val)
	    {
	      ++__i;
	      --__n;
	    }
	  if (__n == 1)
	    return __first;
	  if (__i == __last)
	    return __last;
	  __first = _GLIBCXX_STD_A::find(++__i, __last, __val);
	}
      return __last;
    }
```

#### remove

remove会移除区间内所有与__value相等的元素,这些元素会放到__first和__last两个迭代器区间的后面。

返回为删除之后的结尾迭代器, 即`[__result, __last)`范围的值均为删除的值为__value的节点。

同理, remove_if为删除满足谓词仿函数_Predicate=true的元素。

```cpp
  template<typename _ForwardIterator, typename _Tp>
    _ForwardIterator
    remove(_ForwardIterator __first, _ForwardIterator __last,
	   const _Tp& __value)
    {
      // concept requirements
      __glibcxx_function_requires(_Mutable_ForwardIteratorConcept<
				  _ForwardIterator>)
      __glibcxx_function_requires(_EqualOpConcept<
	    typename iterator_traits<_ForwardIterator>::value_type, _Tp>)
      __glibcxx_requires_valid_range(__first, __last);

      __first = _GLIBCXX_STD_A::find(__first, __last, __value);
      if(__first == __last) // 没有能查找到__value
        return __first;
      _ForwardIterator __result = __first;// __first是第一个指向值为__value节点的迭代器
      ++__first;
      for(; __first != __last; ++__first)
        if(!(*__first == __value))  // first指向节点不等于value
          {
            *__result = _GLIBCXX_MOVE(*__first);  // first位置和result换位, swap
            ++__result;// result位置后移动
          }
      return __result;  // __result 是删除之后的结尾, 即[__result, __last)范围的值均为删除的值为__value的节点
    }

  template<typename _ForwardIterator, typename _Predicate>
    _ForwardIterator
    remove_if(_ForwardIterator __first, _ForwardIterator __last,
	      _Predicate __pred)
    {
      // concept requirements
      __glibcxx_function_requires(_Mutable_ForwardIteratorConcept<
				  _ForwardIterator>)
      __glibcxx_function_requires(_UnaryPredicateConcept<_Predicate,
	    typename iterator_traits<_ForwardIterator>::value_type>)
      __glibcxx_requires_valid_range(__first, __last);

      __first = _GLIBCXX_STD_A::find_if(__first, __last, __pred);
      if(__first == __last)
        return __first;
      _ForwardIterator __result = __first;
      ++__first;
      for(; __first != __last; ++__first)
        if(!bool(__pred(*__first)))
          {
            *__result = _GLIBCXX_MOVE(*__first);
            ++__result;
          }
      return __result;
    }
```

#### unique

unique作用是remove相邻的重复元素, 例如`{1,2,0,3,3,0,7,7,7,0,8}`执行unqiue后变为`{1,2,0,3,0,7,0,8,3,7,7}`。

由于unique只是删除相邻的重复元素, 因此使用前往往先对元素排序。

__unique_copy可以将不重复的元素copy到一个新的迭代器_OutputIterator __result
```cpp
  template<typename _ForwardIterator>
    _ForwardIterator
    unique(_ForwardIterator __first, _ForwardIterator __last)
    {
      // concept requirements
      __glibcxx_function_requires(_Mutable_ForwardIteratorConcept<
				  _ForwardIterator>)
      __glibcxx_function_requires(_EqualityComparableConcept<
		     typename iterator_traits<_ForwardIterator>::value_type>)
      __glibcxx_requires_valid_range(__first, __last);

      // Skip the beginning, if already unique.
      __first = _GLIBCXX_STD_A::adjacent_find(__first, __last);
      if (__first == __last)
	return __last;

      // Do the real copy work.
      _ForwardIterator __dest = __first;
      ++__first;
      while (++__first != __last)
	if (!(*__dest == *__first))
	  *++__dest = _GLIBCXX_MOVE(*__first);  // 
      return ++__dest;
    }


  template<typename _ForwardIterator, typename _OutputIterator>
    _OutputIterator
    __unique_copy(_ForwardIterator __first, _ForwardIterator __last,
		  _OutputIterator __result,
		  forward_iterator_tag, output_iterator_tag)
    {
      // concept requirements -- taken care of in dispatching function
      _ForwardIterator __next = __first;
      *__result = *__first;
      while (++__next != __last)
	if (!(*__first == *__next))
	  {
	    __first = __next;
	    *++__result = *__first; // first是不重复的元素, 赋给__result
	  }
      return ++__result;
    }
```

#### reverse

顾名思义, 将两个_BidirectionalIterator迭代器指向的区间反转。

reverse_copy可以将结果输出到_OutputIterator所属的容器中
```cpp
  template<typename _BidirectionalIterator>
    void
    __reverse(_BidirectionalIterator __first, _BidirectionalIterator __last,
	      bidirectional_iterator_tag)
    {
      while (true)
	if (__first == __last || __first == --__last) // 这里实现了如果__first == __last返回, 否则会--__last。即__first >= __last
	  return;
	else
	  {
	    std::iter_swap(__first, __last);  // 交换迭代器
	    ++__first;
	  }
    }

  template<typename _BidirectionalIterator, typename _OutputIterator>
    _OutputIterator
    reverse_copy(_BidirectionalIterator __first, _BidirectionalIterator __last,
		 _OutputIterator __result)
    {
      // concept requirements
      __glibcxx_function_requires(_BidirectionalIteratorConcept<
				  _BidirectionalIterator>)
      __glibcxx_function_requires(_OutputIteratorConcept<_OutputIterator,
		typename iterator_traits<_BidirectionalIterator>::value_type>)
      __glibcxx_requires_valid_range(__first, __last);

      while (__first != __last)
	{
	  --__last;
	  *__result = *__last;
	  ++__result;
	}
      return __result;
    }
```

#### gcd

求最大公约数, 基于gcd(a, b) = gcd(b, a mod b)
```cpp
  template<typename _EuclideanRingElement>
    _EuclideanRingElement
    __gcd(_EuclideanRingElement __m, _EuclideanRingElement __n)
    {
      while (__n != 0)
	{
	  _EuclideanRingElement __t = __m % __n;
	  __m = __n;
	  __n = __t;
	}
      return __m;
    }
```

#### __rotate

将__first移动到__middle位置, 旋转距离为__middle-__first。利用`std::rotate(std::begin(words), std::begin(words)+3, std::end(words));`, 则坐标更新为`(i+3)%size`

```cpp
  template<typename _ForwardIterator>
    void
    __rotate(_ForwardIterator __first,
	     _ForwardIterator __middle,
	     _ForwardIterator __last,
	     forward_iterator_tag)
    {
      if (__first == __middle || __last  == __middle)
	return;

      _ForwardIterator __first2 = __middle;
      do
	{
	  std::iter_swap(__first, __first2);
	  ++__first;
	  ++__first2;
	  if (__first == __middle)
	    __middle = __first2;
	}
      while (__first2 != __last);

      __first2 = __middle;

      while (__first2 != __last)
	{
	  std::iter_swap(__first, __first2);
	  ++__first;
	  ++__first2;
	  if (__first == __middle)
	    __middle = __first2;
	  else if (__first2 == __last)
	    __first2 = __middle;
	}
    }
// 用户调用接口
  template<typename _ForwardIterator>
    inline void
    rotate(_ForwardIterator __first, _ForwardIterator __middle,
	   _ForwardIterator __last)
    {
      // concept requirements
      __glibcxx_function_requires(_Mutable_ForwardIteratorConcept<
				  _ForwardIterator>)
      __glibcxx_requires_valid_range(__first, __middle);
      __glibcxx_requires_valid_range(__middle, __last);

      typedef typename iterator_traits<_ForwardIterator>::iterator_category
	_IterType;
      std::__rotate(__first, __middle, __last, _IterType());
    }
```

#### count

count用于计数, 顾名思义, 得到迭代器范围的序列等于__value的节点的数量
```cpp
  template<typename _InputIterator, typename _Tp>
    typename iterator_traits<_InputIterator>::difference_type
    count(_InputIterator __first, _InputIterator __last, const _Tp& __value)
    {
      // concept requirements
      __glibcxx_function_requires(_InputIteratorConcept<_InputIterator>)
      __glibcxx_function_requires(_EqualOpConcept<
	typename iterator_traits<_InputIterator>::value_type, _Tp>)
      __glibcxx_requires_valid_range(__first, __last);
      typename iterator_traits<_InputIterator>::difference_type __n = 0;
      for (; __first != __last; ++__first)
	if (*__first == __value)
	  ++__n;
      return __n;
    }
```

#### 洗牌算法 random_shuffle

Knuth-Shuffle洗牌算法`std::iter_swap(__i, __first + (std::rand() % ((__i - __first) + 1)));`

每次在`[0,i]`随机选择一个位置, 与i交换。
```py
for i in range(N):
  j = random.randint(0, i)  # 前闭后闭[a,b]
  a[j], a[i] = a[i], a[j]
```

```cpp
  template<typename _RandomAccessIterator>
    inline void
    random_shuffle(_RandomAccessIterator __first, _RandomAccessIterator __last)
    {
      // concept requirements
      __glibcxx_function_requires(_Mutable_RandomAccessIteratorConcept<
	    _RandomAccessIterator>)
      __glibcxx_requires_valid_range(__first, __last);

      if (__first != __last)
	for (_RandomAccessIterator __i = __first + 1; __i != __last; ++__i)
	  std::iter_swap(__i, __first + (std::rand() % ((__i - __first) + 1))); // 每次__i和随机选的__first + (std::rand() % ((__i - __first) + 1))交换
    }
```

#### 最值max_element, min_element

注意max_element, min_element的Comp都写成operator<形式
```cpp
  template<typename _ForwardIterator, typename _Compare>
    _ForwardIterator
    max_element(_ForwardIterator __first, _ForwardIterator __last,
		_Compare __comp)
    {
      // concept requirements
      __glibcxx_function_requires(_ForwardIteratorConcept<_ForwardIterator>)
      __glibcxx_function_requires(_BinaryPredicateConcept<_Compare,
	    typename iterator_traits<_ForwardIterator>::value_type,
	    typename iterator_traits<_ForwardIterator>::value_type>)
      __glibcxx_requires_valid_range(__first, __last);

      if (__first == __last) return __first;
      _ForwardIterator __result = __first;
      while (++__first != __last)
	if (__comp(*__result, *__first))   // __result < *__first
	  __result = __first;
      return __result;
    }


  template<typename _ForwardIterator, typename _Compare>
    _ForwardIterator
    min_element(_ForwardIterator __first, _ForwardIterator __last,
		_Compare __comp)
    {
      // concept requirements
      __glibcxx_function_requires(_ForwardIteratorConcept<_ForwardIterator>)
      __glibcxx_function_requires(_BinaryPredicateConcept<_Compare,
	    typename iterator_traits<_ForwardIterator>::value_type,
	    typename iterator_traits<_ForwardIterator>::value_type>)
      __glibcxx_requires_valid_range(__first, __last);

      if (__first == __last)
	return __first;
      _ForwardIterator __result = __first;
      while (++__first != __last)
	if (__comp(*__first, *__result))  // *__first < *__result
	  __result = __first;
      return __result;
    }
```

### sort

sort的实现是algorithm一个大的分支, 我们先看用户调用接口
```cpp
  template<typename _RandomAccessIterator>
    inline void
    sort(_RandomAccessIterator __first, _RandomAccessIterator __last)
    {
      // concept requirements
      __glibcxx_function_requires(_Mutable_RandomAccessIteratorConcept<
	    _RandomAccessIterator>)
      __glibcxx_function_requires(_LessThanComparableConcept<
	    typename iterator_traits<_RandomAccessIterator>::value_type>)
      __glibcxx_requires_valid_range(__first, __last);
      __glibcxx_requires_irreflexive(__first, __last);

      std::__sort(__first, __last, __gnu_cxx::__ops::__iter_less_iter());
    }

  template<typename _RandomAccessIterator, typename _Compare>
    inline void
    __sort(_RandomAccessIterator __first, _RandomAccessIterator __last,
	   _Compare __comp)
    {
      if (__first != __last)
	{
	  std::__introsort_loop(__first, __last,
				std::__lg(__last - __first) * 2,
				__comp);
	  std::__final_insertion_sort(__first, __last, __comp);
	}
    }
```

这里有个std::__introsort_loop, 因为快速排序可能因为不当的pivot选择最坏复杂度为O(N^2), std::__introsort_loop正是解决这个问题

1. 在数据量很大时采用正常的快速排序，此时效率为O(logN)。
2. 一旦分段后partition的数据量小于某个阈值，就改用插入排序，因为此时这个分段是基本有序的，这时效率可达O(N)。
3. 在递归过程中，如果递归层次过深，分割行为有恶化倾向时，它能够自动侦测出来，使用堆排序来处理，在此情况下，使其效率维持在堆排序的O(N logN)，但这又比一开始使用堆排序好。

选择的最大深度是`log2(__last - __first) * 2`, 一旦超过这个深度, 调用堆排序。

下面是std::_introsort_loop的实现, 值得注意的是快速排序以迭代的形式写
```cpp
  template<typename _RandomAccessIterator, typename _Size, typename _Compare>
    void
    __introsort_loop(_RandomAccessIterator __first,
		     _RandomAccessIterator __last,
		     _Size __depth_limit, _Compare __comp)
    {
      while (__last - __first > int(_S_threshold))  // _S_threshold设置为16, 小于16的片段不执行快排
	{
	  if (__depth_limit == 0)  // 递归深度为0, 调用partial_sort, 实际是对这一个片段使用堆排序
	    {
	      _GLIBCXX_STD_A::partial_sort(__first, __last, __last, __comp);  
	      return;
	    }
	  --__depth_limit;
	  _RandomAccessIterator __cut =
	    std::__unguarded_partition_pivot(__first, __last, __comp);  // 快速排序的分片
	  std::__introsort_loop(__cut, __last, __depth_limit, __comp);
	  __last = __cut; // 递归
	}
    }
```

partial_sort用意是Sort the smallest elements of a sequence using a predicate for comparison, 实际实现是堆排序
```cpp
  template<typename _RandomAccessIterator>
    inline void
    partial_sort(_RandomAccessIterator __first,
		 _RandomAccessIterator __middle,
		 _RandomAccessIterator __last)
    {
      typedef typename iterator_traits<_RandomAccessIterator>::value_type
	_ValueType;

      // concept requirements
      __glibcxx_function_requires(_Mutable_RandomAccessIteratorConcept<
	    _RandomAccessIterator>)
      __glibcxx_function_requires(_LessThanComparableConcept<_ValueType>)
      __glibcxx_requires_valid_range(__first, __middle);
      __glibcxx_requires_valid_range(__middle, __last);

      std::__heap_select(__first, __middle, __last);  // 建堆
      std::sort_heap(__first, __middle);  // 将最大值移到堆尾, 调整堆
    }

  template<typename _RandomAccessIterator>
    void
    __heap_select(_RandomAccessIterator __first,
		  _RandomAccessIterator __middle,
		  _RandomAccessIterator __last)
    {
      std::make_heap(__first, __middle);
      for (_RandomAccessIterator __i = __middle; __i < __last; ++__i)
	if (*__i < *__first)
	  std::__pop_heap(__first, __middle, __i);
    }

  template<typename _RandomAccessIterator, typename _Compare>
    void
    __sort_heap(_RandomAccessIterator __first, _RandomAccessIterator __last,
		_Compare& __comp)
    {
      while (__last - __first > 1)
	{
	  --__last;
	  std::__pop_heap(__first, __last, __last, __comp);
	}
    }
```

__unguarded_partition_pivot的实现, 实际是快排的分片. 选取pivot通过**三点中值法**, 也就是__median函数，它的作用是取首部、尾部和中部三个元素的中值作为pivot, 而非一般使用的第一个元素
```cpp
  template<typename _RandomAccessIterator, typename _Compare>
    inline _RandomAccessIterator
    __unguarded_partition_pivot(_RandomAccessIterator __first,
				_RandomAccessIterator __last, _Compare __comp)
    {
      _RandomAccessIterator __mid = __first + (__last - __first) / 2;
      std::__move_median_first(__first, __mid, (__last - 1), __comp);
      return std::__unguarded_partition(__first + 1, __last, *__first, __comp); // *__first元素作为partition的pivot
    }

  // 分片partition
  template<typename _RandomAccessIterator, typename _Tp, typename _Compare>
    _RandomAccessIterator
    __unguarded_partition(_RandomAccessIterator __first,
			  _RandomAccessIterator __last,
			  const _Tp& __pivot, _Compare __comp)
    {
      while (true)
	{
	  while (__comp(*__first, __pivot)) // 从小到大排序, 则直到满足*first >= __pivot
	    ++__first;
	  --__last;
	  while (__comp(__pivot, *__last))
	    --__last;
	  if (!(__first < __last))
	    return __first;
	  std::iter_swap(__first, __last);  // first last交换
	  ++__first;
	}
    }
```

注意__unguarded_partition的实现, `__comp(*__first, __pivot)`, 对于全等的序列, 例如{1,1,1,1}, 如果我们的__comp设置为`a <= b`, 这样__comp就会一直是true, __first迭代器就会一直++直到越界(事实上越界它也不会停止), 这就带来风险。**因此`__comp`写法需要严格保证`<`,`>`, 或者`==`, 不要有`>=`, `<=` 这种等式逻辑称之为严格弱序**。

`__comp`设置`a<b`效率处理带有大量重复数字的序列例如{1,1,1,1}, 是比较低的, 因为它只是通过`--last`,`++first`直到`!(__first < __last)`返回, 效率是O(Nlog(N))。处理大量重复数字排序当基本有序时可以使用插入排序, 效率可达O(N), 这也是sort采用快排+插入排序结合的原因。


__final_insertion_sort, 快排/堆排前提是片段大于_S_threshold, 即16。这样快排过后__introsort_loop并没有完全排完, 但是可以保证大多数有序。所以接下来就是插入排序处理剩下的了。当有序的情况下插入排序效率是O(N), 优于快排。

```cpp
  template<typename _RandomAccessIterator, typename _Compare>
    void
    __final_insertion_sort(_RandomAccessIterator __first,
			   _RandomAccessIterator __last, _Compare __comp)
    {
      if (__last - __first > int(_S_threshold))
	{
	  std::__insertion_sort(__first, __first + int(_S_threshold), __comp);
	  std::__unguarded_insertion_sort(__first + int(_S_threshold), __last,
					  __comp);
	}
      else
	std::__insertion_sort(__first, __last, __comp);
    }
```

插入排序
```cpp
  template<typename _RandomAccessIterator, typename _Compare>
    void
    __insertion_sort(_RandomAccessIterator __first,
		     _RandomAccessIterator __last, _Compare __comp)
    {
      if (__first == __last) return;

      for (_RandomAccessIterator __i = __first + 1; __i != __last; ++__i)
	{
	  if (__comp(*__i, *__first)) // *__i < *__first, *__i应该插入到*__first的前面
	    {
	      typename iterator_traits<_RandomAccessIterator>::value_type
		__val = _GLIBCXX_MOVE(*__i);
	      _GLIBCXX_MOVE_BACKWARD3(__first, __i, __i + 1); // _first位置统一后移动1位
	      *__first = _GLIBCXX_MOVE(__val);  // 原先的_first位置用__i代替
	    }
	  else
	    std::__unguarded_linear_insert(__i, __comp);
	}
    }
```


#### nth_element

值得注意的是, 寻找第N大的数也可以用快速排序找。每次partition后pivot的位置index可以认为pivot为第index大的数, 基于此更改范围可以找第N大的数。

nth_dement() 的执行会导致第 n 个元素被放置在适当的位置。这个范围内，在第 n 个元素之前的元素都小于第 n 个元素，而且它后面的每个元素都会比它大。注意nth_dement()是侵入式的, 返回为void。而一般我们要求更多是容器不变返回第n大的数。

```cpp
  template<typename _RandomAccessIterator, typename _Compare>
    inline void
    nth_element(_RandomAccessIterator __first, _RandomAccessIterator __nth,
		_RandomAccessIterator __last, _Compare __comp)
    {
      typedef typename iterator_traits<_RandomAccessIterator>::value_type
	_ValueType;

      // concept requirements
      __glibcxx_function_requires(_Mutable_RandomAccessIteratorConcept<
				  _RandomAccessIterator>)
      __glibcxx_function_requires(_BinaryPredicateConcept<_Compare,
				  _ValueType, _ValueType>)
      __glibcxx_requires_valid_range(__first, __nth);
      __glibcxx_requires_valid_range(__nth, __last);

      if (__first == __last || __nth == __last)
	return;

      std::__introselect(__first, __nth, __last,
			 std::__lg(__last - __first) * 2, __comp);
    }
```

示例
```cpp
int main() {

	int array[] = {5,6,7,8,4,2,1,3,0};
	int len=sizeof(array)/sizeof(int);
	cout<<"排序前: ";
	for(int i=0; i<len; i++)
		cout<<array[i]<<" ";
	nth_element(array, array+6, array+len);  //排序第6个元素
		cout<<endl;
	cout<<"排序后："; 
	for(int i=0; i<len; i++)
		cout<<array[i]<<" ";
		
	cout<<endl<<"第6个元素为"<<array[6]<<endl; 
}

输出
排序前: 5 6 7 8 4 2 1 3 0 
排序后：4 0 3 1 2 5 6 7 8 
第6个元素为6
```

### stable_sort

sort算法基于快速排序, 插入排序, 堆排序的adaptor, 其中快速排序和堆排序都是不稳定排序。

显然stable_sort应该基于merge_sort, 基本思路是建立一个缓冲区, 如果数据量太大则先分块进行归并排序, 最后再通过插入排序将块合并。

```cpp
  template<typename _RandomAccessIterator, typename _Compare>
    inline void
    stable_sort(_RandomAccessIterator __first, _RandomAccessIterator __last,
		_Compare __comp)
    {
      typedef typename iterator_traits<_RandomAccessIterator>::value_type
	_ValueType;
      typedef typename iterator_traits<_RandomAccessIterator>::difference_type
	_DistanceType;

      // concept requirements
      __glibcxx_function_requires(_Mutable_RandomAccessIteratorConcept<
	    _RandomAccessIterator>)
      __glibcxx_function_requires(_BinaryPredicateConcept<_Compare,
				  _ValueType,
				  _ValueType>)
      __glibcxx_requires_valid_range(__first, __last);

      _Temporary_buffer<_RandomAccessIterator, _ValueType> __buf(__first,
								 __last);   // 临时的空间
      if (__buf.begin() == 0)
	std::__inplace_stable_sort(__first, __last, __comp);
      else
	std::__stable_sort_adaptive(__first, __last, __buf.begin(),
				    _DistanceType(__buf.size()), __comp);
    }


  template<typename _RandomAccessIterator, typename _Pointer,
	   typename _Distance, typename _Compare>
    void
    __stable_sort_adaptive(_RandomAccessIterator __first,
			   _RandomAccessIterator __last,
                           _Pointer __buffer, _Distance __buffer_size,
                           _Compare __comp)
    {
      const _Distance __len = (__last - __first + 1) / 2;
      const _RandomAccessIterator __middle = __first + __len;
      if (__len > __buffer_size)
	{
	  std::__stable_sort_adaptive(__first, __middle, __buffer,
				      __buffer_size, __comp);
	  std::__stable_sort_adaptive(__middle, __last, __buffer,
				      __buffer_size, __comp);
	}
      else
	{
	  std::__merge_sort_with_buffer(__first, __middle, __buffer, __comp);
	  std::__merge_sort_with_buffer(__middle, __last, __buffer, __comp);
	}
      std::__merge_adaptive(__first, __middle, __last,
			    _Distance(__middle - __first),
			    _Distance(__last - __middle),
			    __buffer, __buffer_size,
			    __comp);
    }
```

归并排序

```cpp
  template<typename _RandomAccessIterator, typename _Pointer>
    void
    __merge_sort_with_buffer(_RandomAccessIterator __first,
			     _RandomAccessIterator __last,
                             _Pointer __buffer)
    {
      typedef typename iterator_traits<_RandomAccessIterator>::difference_type
	_Distance;

      const _Distance __len = __last - __first;
      const _Pointer __buffer_last = __buffer + __len;

      _Distance __step_size = _S_chunk_size;
      std::__chunk_insertion_sort(__first, __last, __step_size);    // 块插入排序

      while (__step_size < __len)
	{
	  std::__merge_sort_loop(__first, __last, __buffer, __step_size);
	  __step_size *= 2;
	  std::__merge_sort_loop(__buffer, __buffer_last, __first, __step_size);
	  __step_size *= 2;
	}
    }

  template<typename _RandomAccessIterator1, typename _RandomAccessIterator2,
	   typename _Distance>
    void
    __merge_sort_loop(_RandomAccessIterator1 __first,
		      _RandomAccessIterator1 __last,
		      _RandomAccessIterator2 __result,
		      _Distance __step_size)
    {
      const _Distance __two_step = 2 * __step_size;

      while (__last - __first >= __two_step)
	{
	  __result = std::__move_merge(__first, __first + __step_size,
				       __first + __step_size,
				       __first + __two_step, __result);
	  __first += __two_step;
	}

      __step_size = std::min(_Distance(__last - __first), __step_size);
      std::__move_merge(__first, __first + __step_size,
			__first + __step_size, __last, __result);
    }

// 块插入排序, 在[__first,__last)区间分布着很多__chunk_size的块, 块内有序
  template<typename _RandomAccessIterator, typename _Distance,
	   typename _Compare>
    void
    __chunk_insertion_sort(_RandomAccessIterator __first,
			   _RandomAccessIterator __last,
			   _Distance __chunk_size, _Compare __comp)
    {
      while (__last - __first >= __chunk_size)
	{
	  std::__insertion_sort(__first, __first + __chunk_size, __comp);
	  __first += __chunk_size;
	}
      std::__insertion_sort(__first, __last, __comp);
    }
```

merge
```cpp
  template<typename _InputIterator1, typename _InputIterator2,
	   typename _OutputIterator, typename _Compare>
    _OutputIterator
    merge(_InputIterator1 __first1, _InputIterator1 __last1,
	  _InputIterator2 __first2, _InputIterator2 __last2,
	  _OutputIterator __result, _Compare __comp)
    {
      typedef typename iterator_traits<_InputIterator1>::value_type
	_ValueType1;
      typedef typename iterator_traits<_InputIterator2>::value_type
	_ValueType2;

      // concept requirements
      __glibcxx_function_requires(_InputIteratorConcept<_InputIterator1>)
      __glibcxx_function_requires(_InputIteratorConcept<_InputIterator2>)
      __glibcxx_function_requires(_OutputIteratorConcept<_OutputIterator,
				  _ValueType1>)
      __glibcxx_function_requires(_OutputIteratorConcept<_OutputIterator,
				  _ValueType2>)
      __glibcxx_function_requires(_BinaryPredicateConcept<_Compare,
				  _ValueType2, _ValueType1>)
      __glibcxx_requires_sorted_set_pred(__first1, __last1, __first2, __comp);
      __glibcxx_requires_sorted_set_pred(__first2, __last2, __first1, __comp);

      while (__first1 != __last1 && __first2 != __last2)
	{
	  if (__comp(*__first2, *__first1))
	    {
	      *__result = *__first2;
	      ++__first2;
	    }
	  else
	    {
	      *__result = *__first1;
	      ++__first1;
	    }
	  ++__result;
	}
      return std::copy(__first2, __last2, std::copy(__first1, __last1,
						    __result));  // std::copy, 将某个迭代器指向的元素拷贝到另一个迭代器处
    }
```

### 二分查找

#### lower_bound和upper_bound

* lower_bound

如果迭代器范围`[_ForwardIterator __first, _ForwardIterator __last)`的元素是有序的。

如果从小到大排序, _Compare为<, 那么将找到第一个大于等于__val的元素, 返回迭代器。如果是从大到小排序, 那么找到第一个小于等于__val的元素, 返回迭代器

```cpp
      while (__len > 0)
	{
	  _DistanceType __half = __len >> 1;  // 折半
	  _ForwardIterator __middle = __first;
	  std::advance(__middle, __half); // 移动到__middle
	  if (__comp(*__middle, __val)) // __middle < __val
	    {
	      __first = __middle;
	      ++__first;  // 因为__middle < __val, 所以__first=__middle+1, 不把middle算在内
	      __len = __len - __half - 1;
	    }
	  else    // __middle >= __val
	    __len = __half; // 要找的位于[first, __middle], 这里把middle也算在内
	}
      return __first;
```

* upper_bound

与lower_bound实现, 类似

如果从小到大排序, _Compare为<, 那么将找到第一个大于__val的元素, 返回迭代器。如果是从大到小排序, 那么找到第一个小于__val的元素, 返回迭代器。

值得注意的是, lower_bound和upper_bound二分查找的实现基于__first迭代器和len长度, 而不是我们常用的start, end两个位置。__middle就是__first+__len>>1; 因为迭代器是_ForwardIterator, 也就是只能前行不能后移, 基于迭代器继承关系, 红黑树和vector, list都可以使用lower_bound和upper_bound。


注意lower_bound和upper_bound要想收敛, 就必须一种情况把middle算在内, 例如lower_bound目的是找>=val的元素, 那么如果__middle >= val, 也就是`!__comp(*__middle, __val)`, 就得把middle算在内, 也就是` __len = __half`; 而upper_bound目的是找>val的元素, 那么__middle > val 才能把middle算在内, 也就是`__comp(__val, *__middle)`, 有`__len = __half`

```cpp
      while (__len > 0)
	{
	  _DistanceType __half = __len >> 1;  // 折半查找
	  _ForwardIterator __middle = __first;
	  std::advance(__middle, __half);
	  if (__comp(__val, *__middle)) // 如果_val < *_middle
	    __len = __half; // 范围为[first, middle], middle被算在内
	  else    // _val >= *_middle
	    {
	      __first = __middle;
	      ++__first;  // __first=__middle+1, __middle不算在内
	      __len = __len - __half - 1; // 不算middle
	    }
	}
      return __first;
```

lower_bound和upper_bound源码
```cpp
  template<typename _ForwardIterator, typename _Tp, typename _Compare>
    _ForwardIterator
    lower_bound(_ForwardIterator __first, _ForwardIterator __last,
		const _Tp& __val, _Compare __comp)
    {
      typedef typename iterator_traits<_ForwardIterator>::value_type
	_ValueType;
      typedef typename iterator_traits<_ForwardIterator>::difference_type
	_DistanceType;

      // concept requirements
      __glibcxx_function_requires(_ForwardIteratorConcept<_ForwardIterator>)
      __glibcxx_function_requires(_BinaryPredicateConcept<_Compare,
				  _ValueType, _Tp>)
      __glibcxx_requires_partitioned_lower_pred(__first, __last,
						__val, __comp);

      _DistanceType __len = std::distance(__first, __last);

      while (__len > 0)
	{
	  _DistanceType __half = __len >> 1;  // 折半
	  _ForwardIterator __middle = __first;
	  std::advance(__middle, __half); // 移动到__middle
	  if (__comp(*__middle, __val)) // __middle < __val
	    {
	      __first = __middle;
	      ++__first;  // __first前进, 可以>=
	      __len = __len - __half - 1;
	    }
	  else
	    __len = __half;
	}
      return __first;
    }


  template<typename _ForwardIterator, typename _Tp, typename _Compare>
    _ForwardIterator
    upper_bound(_ForwardIterator __first, _ForwardIterator __last,
		const _Tp& __val, _Compare __comp)
    {
      typedef typename iterator_traits<_ForwardIterator>::value_type
	_ValueType;
      typedef typename iterator_traits<_ForwardIterator>::difference_type
	_DistanceType;

      // concept requirements
      __glibcxx_function_requires(_ForwardIteratorConcept<_ForwardIterator>)
      __glibcxx_function_requires(_BinaryPredicateConcept<_Compare,
				  _Tp, _ValueType>)
      __glibcxx_requires_partitioned_upper_pred(__first, __last,
						__val, __comp);

      _DistanceType __len = std::distance(__first, __last);

      while (__len > 0)
	{
	  _DistanceType __half = __len >> 1;  // 折半查找
	  _ForwardIterator __middle = __first;
	  std::advance(__middle, __half);
	  if (__comp(__val, *__middle)) 
	    __len = __half;
	  else
	    {
	      __first = __middle;
	      ++__first;
	      __len = __len - __half - 1;
	    }
	}
      return __first;
    }
```

* binarysearch

这个二分查找基于lower_bound, 返回bool; 一般我们用二分查找都是返回迭代器, 因此基本不会用这个
```cpp
  template<typename _ForwardIterator, typename _Tp>
    bool
    binary_search(_ForwardIterator __first, _ForwardIterator __last,
                  const _Tp& __val)
    {
      typedef typename iterator_traits<_ForwardIterator>::value_type
	_ValueType;

      _ForwardIterator __i = std::lower_bound(__first, __last, __val);  // __i是>=__val的迭代器
      return __i != __last && !(__val < *__i);  
    }
```

### numberic

numberic的算法位于`#include <numberic>`中, 但有的也很常用

* accumulate

求数字序列的和, 注意参数`_Tp __init`作用是决定返回类型, 设置为0返回类型为int, 设置为0.0则返回类型为double
```cpp
  template<typename _InputIterator, typename _Tp>
    inline _Tp
    accumulate(_InputIterator __first, _InputIterator __last, _Tp __init)
    {
      // concept requirements
      __glibcxx_function_requires(_InputIteratorConcept<_InputIterator>)
      __glibcxx_requires_valid_range(__first, __last);

      for (; __first != __last; ++__first)
	__init = __init + *__first;
      return __init;
    }
```

* inner_product

同样需要__init决定返回类型
```cpp
  template<typename _InputIterator1, typename _InputIterator2, typename _Tp>
    inline _Tp
    inner_product(_InputIterator1 __first1, _InputIterator1 __last1,
		  _InputIterator2 __first2, _Tp __init)
    {
      // concept requirements
      __glibcxx_function_requires(_InputIteratorConcept<_InputIterator1>)
      __glibcxx_function_requires(_InputIteratorConcept<_InputIterator2>)
      __glibcxx_requires_valid_range(__first1, __last1);

      for (; __first1 != __last1; ++__first1, ++__first2)
	__init = __init + (*__first1 * *__first2);
      return __init;
    }
```