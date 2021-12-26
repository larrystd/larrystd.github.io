---
title: 标准库chrono和future
date: 2021-12-08
updated: 2021-12-08
tags: 
  - C++标准库
  - cpp
categories:
  - source code
aplayer: true
---

> 基于g++ 4.8, 读源码是为了更好了使用标准库

### chrono

`#include <chrono>`是C++的是时间库, 主要有3个概念，分别是持续时间（Duration）、时间点（timepoint）、时钟（clock）

#### clock 时钟

定义中的时钟主要有3个，分别是:

1. system_clock，顾名思义，就是系统时钟，他一般是unix时间，即从1970年1月1日到现在的间隔。如果系统时间发生调整，那么调用该方法获取的时间值也会跟着变化。
2. steady_clock，是单调时间，即每后一次调用都一定会比前一次调用的时间要晚，它并不反映真实的世界时间。在内部的实现中，有可能是系统开启以来的时间。通过该时钟很适合用来获取一段时间内的间隔，它不随系统时间改变而改变。
3. high_resolution_clock，高精度时间，用来测量细微间隔的一段时间，它的内部实现中可能是system_clock或是steady_clock，也可能是其他独立的时间。
一般来说，我们使用的都是system_clock。system_clock的主要方法有3个，分别是：
```
now，用来获取当前时间。
to_time_t，用来将系统时间转变为std::time_t类型。
from_time_t，用来将std::time_t类型转换为系统时间点。
```

```cpp
#include<ctime>//将时间格式的数据转换成字符串
#include<chrono>
using namespace std::chrono;
using namespace std;
int main()
{
    //获取系统的当前时间
    auto t = system_clock::now();
    //将获取的时间转换成time_t类型
    auto tNow = system_clock::to_time_t(t);

    //ctime()函数将time_t类型的时间转化成字符串格式,这个字符串自带换行符
    string str_time = std::ctime(&tNow);
    cout<<str_time;
}
Wed Dec  8 15:38:55 2021
```

#### Duration持续时间

持续时间可以通过以下构建
```
std::chrono::nanoseconds	
std::chrono::microseconds	
std::chrono::milliseconds	
std::chrono::seconds	
std::chrono::minutes
std::chrono::hours
```

```cpp
   // DEMO:通过获取两个时刻的时间，然后计算时间长度

    using namespace std::chrono;

    // Step one: 定义一个clock
    typedef system_clock sys_clk_t;

    // Step two: 分别获取两个时刻的时间
    typedef system_clock::time_point time_point_t;
    // 第1个时间
    time_point_t time01 = sys_clk_t::now();
    // 延时5秒
    std::this_thread::sleep_for(std::chrono::duration<int>(5));
    // 第2个时间
    time_point_t time02 = sys_clk_t::now();

    // Step three: 计算时间差
    cout << "dt_time(system_clock period) = " << (time02 - time01).count() << endl;

dt_time(system_clock period) = 5000469400
```
<!-- more -->
#### future

