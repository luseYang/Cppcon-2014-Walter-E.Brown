---
title: 现代模版元编程
tags: ["template"]
categories: [Cppcon 2014 - Walter E. Brown]
description: '深挖 C++ 模版知识'
date: 2024-09-16 21:44:21
cover: https://s2.loli.net/2024/09/16/NU7CT6YbWmAcDfu.png
---

# 前言

模板元编程已成为 C++ 程序员工具包的重要组成部分。本文将学习并展示元编程工具和技术，并应用于每个标准库设施的代表实现。在此过程中，我们将研究 `void_t`，这是最近提出的一种极其简单的新 `<type_traits>` 候选项，一位专家将其描述为 “高度先进(且优雅)，甚至对经验丰富的模板元编程人员来说都令人惊讶。”

> 注意：本文不适合 C++ 新手
> 本文可能存在一些对计算机软件编程方法持有相当强烈的观点，这些观点不是所有程序员都度认同的。但他们应该确实如此

From the `std::` library:
- integral_constant, true_type, false_type;
- is_same, is_void;
- is_copy_assignable, is_move_assignable;
- remove_const, remove_volatile, remove_cv;
- conditional, enable_if;
- is_intergal, is_floating_point;

Not int the `std::` library:
- abs, gcd;
- type_is, bool_constant;
- is_one_of;
- void_t **(!)**, haas_type_member;

# 什么是模版元编程

模板元编程（英语：Template metaprogramming，缩写：TMP）是一种元编程技术，编译器使用模板产生暂时性的源码，然后再和剩下的源码混合并编译。这些模板的输出包括编译时期常量、数据结构以及完整的函数。如此利用模板可以被想成编译期的执行。这种技术被许多语言使用，最为知名的当属C++，其他还有Curl、D、Eiffel，以及语言扩展，如Template Haskell。                           

————来自维基百科

C++ 模版元编程使用模版实例化来驱动编译时评估.以防你虽然不是 C++ 新手，但不是很熟悉模板，这里举一个例子：*调用 `f(x)`， 其中 `f` 命名一个函数模版，意思就是当我们使用一个模版的名称，其中包含一个(函数，类型，变量)时，编译器将从该模版实例化(创建)预期的实体*。模板元程序员利用这种机制来提高源代码的灵活性和运行时性能，显而易见的，你把运行期做的工作一部分转移到了编译器实现。也正因如此，他从来不是没有代价的，因为编译器正在工作，所以要花费一点时间在编译期。

以防不相信模版带来的性能提升，再举一个例子：`std` 标准库中的 `std::pow()` 函数，将其改为手动实现的模版 `pow<>`(编译器计算)，即使是在 2001 年那种粗糙的计时条件下，计算 x^50 重复 10000000 次。

在700MHz 的奔腾芯片上使用 gcc 2.95.2，-O0，Win2K.

|        | std::pow() | pow<> v1  |  pow<> v2  |
|--------|------------|-----------|------------|
|  real  |  11.858 s  |  8.081 s  |  3.035 s   |
| *user* | *11.837 s* | *8.081 s* | *3.024 s*  |
|  sys   |  0.020 s   |  0.020 s  |  0.030 s   |


使用 g++ -O2 编译，其他方面相同。

|        | std::pow() | pow<> v1  |  pow<> v2  |
|--------|------------|-----------|------------|
|  real  |  11.857 s  |  4.286 s  |  0.300 s   |
| *user* | *11.847 s* | *4.236 s* | *0.190 s*  |
|  sys   |  0.020 s   |  0.010 s  |  0.010 s   |

改进：v1的改进率高达47%，v2的改进率达到了90%!

元编程时：

记住，运行时间 == 编译时间，所以不能依赖：可变性、`virtual`、其他 RTTI 等。所以接下来的讲解均不涉及运行期的任何事情。

# 如何将工作转移到编译期

把工作从运行期转移到编译期并不困难，简单来说就是：当你提前知道你输入的值是什么，如果在运行期前没有这些值，那么元编程不会帮到你。

Example：一个编译器绝对值元函数：

```cpp
template<int N> 
struct abs{
    static_assert(N != INT_MIN);    // C++17-style guard
    static constexpr int value = (N < 0) > -N : N;  // "return"
};
```

这里有一个静态断言有两个原因： 一个是如果是二进制的补码，你不能得到 `INT_MIN` 的绝对值，因为没有正的对应值，二是这是静态断言的变体，被选入了 C++17，在 C++11 中，你必须输入另一个字符串字面量，在 C++17 中这就是可选的(为什么要这么做呢？开发者的回答是：大多数时候不在乎，字符串字面值对我来说至少是引用，因为我从我的编译器中得到诊断包含断言的文本。简而言之，知道错了就好没必要用字符串描述，笑~)。 

又一个有趣的点是：在结尾使用结果进行初始化，这里不是赋值，赋值是运行时的，我们正在初始化一个 `static`，现在更多的是 `constexpr` 变量，这就替代了我们所认为的返回，这是可行的。

Usage(元函数调用)：

Metafunction arg(s) 作为模板的 arg(s) 提供。“调用” 语法是对模版的已发布值的请求。

```cpp
`int const n = ...;     // 也可以是 constexpr
...abs<n>::value ...    // 实例化产生编译时常量。
```

这种写法是自 C++98 起就有用的写法。稍后会介绍在 C++11 14 17 中的其他写法。

# C++11 的 constexpr 函数

Example： 一个编译期求绝对值函数：

```cpp
constexpr int abs(int N) {return (N < 0) ? -N : N;}
```

`constexpr` 函数和元函数是什么关系呢?

`constexpr` 函数的主要优势是它那熟悉的调用语法，没有冒号和尖括号的尴尬，你只要假装它是一个正常函数，但是如果你把元函数和 `constexpr` 函数比较，元函数相当于一个结构体，它给了我们更多的工具，因为你可以在结构体内部将东西公开，你不能在 `constexpr` 函数中做同样的事情。

`constexpr` 函数主要用于编译期的计算，它处理的是具体的数值或数据，可以在编译时计算出函数结果。模板则是为不同的类型生成不同的代码，主要用于处理类型逻辑、元编程等。例如，可以使用模板生成不同的类或函数，而这在 `constexpr` 函数中无法完成。

# 实现编译器递归

Example：`gcd(m，n)` 元函数(编译时计算最大公因数)的模版
 
```cpp
template<unsigned M, unsigned N>
struct gcd{
    static int const value = gcd<N, M % N>::value;
};
```

与模式匹配非常类似，这部分的全特化可以识别 `gcd(m，0)`

```cpp
template<unsigned M>
struct gcd<M, 0>{
    static_assert{M != 0};
    static int const value = M;
};
```

(这里没写 `constexpr` 是因为开发者写这个例子的时候还没有 `constexpr` 呢，不过这样就适用于较老的编译器，随你喜欢)

# 元函数可以使用类型作为参数

如标题所说的，使用 `constexpr` 函数是做不到的，类型不是值。

现在我们就有一个很好的例子`sizeof`，`sizeof` 是一个内置的类型函数，但是我们也可以自己编写。

```cpp
template<class T>
struct rank { static size_t const value = 0u; };
```

这是一个标准库之外的简单例子，用来求数组的维数。就像大多数情况一样它分为两个部分，这是第一部分，如果传入的类型不是一个数组，这个元函数就返回 0，

```cpp
template<class U, size_t N>
struct rank<U[N]>{
    static size_t const value = 1u + rank<U>::value;
};
```

Usage:

```cpp
using array_t = int [10][20][30];
...rank<array_t>::value...      // 编译器产出 3u
```

详解：
```cpp
#include <iostream>
#include <memory>

template<class T>
struct Rank
{
	static const size_t value = 0U;
};

/* First instantiated from: insights.cpp:10 */
#ifdef INSIGHTS_USE_TEMPLATE
template<>
struct Rank<int[20][30]>
{
	static const size_t value = 1U + Rank<int[30]>::value;
};

#endif
/* First instantiated from: insights.cpp:10 */
#ifdef INSIGHTS_USE_TEMPLATE
template<>
struct Rank<int[30]>
{
	static const size_t value = 1U + Rank<int>::value;
};

#endif
/* First instantiated from: insights.cpp:10 */
#ifdef INSIGHTS_USE_TEMPLATE
template<>
struct Rank<int>
{
	static const size_t value = 0U;
};

#endif
/* First instantiated from: insights.cpp:15 */
#ifdef INSIGHTS_USE_TEMPLATE
template<>
struct Rank<int[10][20][30]>
{
	static const size_t value = 1U + Rank<int[20][30]>::value;
};

#endif

template<class U, size_t N>
struct Rank<U[N]>
{
	static const size_t value = 1U + Rank<U>::value;
};


int main()
{
	using arr = int[10][20][30];
	std::cout.operator<<(Rank<int[10][20][30]>::value);
	return 0;
}
```

# 元函数也可以作为其结果产生一种类型

Example："remove" 一个类型的 `const-qualification`(不是实际的移除，相当于给我一个等效的没有 `const` 的类型)

```cpp
template<class T>
struct{
    struct remove_const {using type = T; }
};

template<class U>
struct{
    struct remove_const<U const> {using type = U; }
};
```

Usages：

```cpp
remove_const<T>::type t;    // 确保 t 具有可变类型
remove_const_t<T> t;        // C++14 之后的，使用 _t 避免 ::type
```

# C++11 库元函数约定

第一个约定是用于元函数的是使其具有类型结果，正如之前看到的，返回结果的类型用 `type` 给他命名，(除此约定之前的几个 `std::` 元函数外，例如：`iterator_traits` 有5个类型结果，没有一个是命名类型)

Example：身份认同元函数(identity metafunction)

```cpp
template<class T>
struct type_is {
    using type = T;
};
```

现在可以通过继承来应用约定

```cpp
// 主模板处理非易失性限定的类型:
template<class T>
struct remove_volatile : type_is<T> { };

// 部分特化化识别易变限定类型:
template<class U>
struct remove_volatile<U volatile> : type_is<U> { };
```

# 编译时决策

假设一个元函数，IF/IF_t，选择一下两种类型之一

```cpp
template<bool p, class T, class F>
struct IF : type_is< ... > { };       // p ? T : F
```

这样一个工具可以让我们编写自配置代码:

```cpp
int const q = ...;      //用户的配置参数

IF_t<(q < 0), int, unsigned> k;         // 如果 p 是负数，把 k 声明为 int, 否则声明为 unsigned

IF_t<(q < 0), F, G> {}();       // 实例化并调用这两个函数对象中的一个

class D : public IF_t<(q < 0), B1, B2> {...}        // 从这两个基类中的一个继承
```

# IF 的幕后

实现起来也是惊人的简单的:

```cpp
template<bool, class T, calss>      // 无需命名未使用的参数
struct IF : type_is<T> { };

// 先是一个主模版，然后是各个模版特化

// 模版特化处理 false 的情况
template<class T, class F>
struct IF<false, T, F> : type_is<F> { };
```

*这个 `IF` 在 C++11 中被命名为 `conditional`*,由一个方便的调用别名，`conditional_t` 来增强(C++14)。(所有 C++14 标准类型返回特性都具有类似的 `...-t` 方便元函数调用别名。)

# 条件变量的单类型变异

如果为真，则使用给定的类型，如果为假，则不使用任何类型。

```cpp
// 主模板假设 bool 值为真。
template<bool, class T = void>      // 默认是有用的，不是必须的
struct enable_if : type_is<T> { };

// 局部特化识别一个 false 不计算任何值
template<class T>
struct enable_if<false, T> { };
```

这有什么好处呢，有什么用处呢？现在考虑一个元函数调用：

```cpp
enable_if<false, ...>::type;
```

如果 `enable_if` 是 `false`，则启用特化版本，这样的话 `::type` 就什么都不会得到，没有这个 `type` 成员，这是一个错误。当然了，仅在少数情况下是一个错误，因为我们还有 SFINAE！(通常称为显式过载集管理。代换失败不是错误)

# SFINAE 在隐式实例化过程中适用

在模板实例化过程中，编译器将会发生以下操作：
1. 获取模版参数：
- 如果在模版使用是明确提供，则为逐字记录
- 否则是调用点的函数参数中推导出来的
- 否则取自声明的默认模版参数
**在同一个实例化中做这三件事**
2. 将整个模版中的每个参数替换为相应的参数

如果这些步骤生成了格式良好的代码，实例化就成功了，但是，如果生成的代码格式不正确，则被认为是不可行的(由于替换失败，会抛弃之，继续寻找下一个)

# 使用 SFINAE

Example：需要一个算法 `f` 取代积分类型 T，并用第二个 `f` 取代浮点类型 `T` 来加它。

对于给定类型 `T`，最少要实例化两种算法中的一种:

```cpp
template<class T>
enable_if<is_integral<T>::value, maxint_t> f(T val) { ... };

template<class T>
enable_if_t<is_floating_point<T>::value, long double> f(T val) { ... };
```

如果两个重载都可不行

例如，用字符串参数调用 `f` 会产生一个不成型的程序，因为两个候选人都将被 SFINAE 掉。就会得到一个编译错误，没有匹配的模版实例。

# C++11 库元函数约定#2

具有值结果的元函数具有：
- 一个 `static constexpr` 成员，值，给出其结果...
- 几个方便的成员类型和 `constexpr` 函数

经典的 C++11 返回值的元函数

```cpp
template<class T, T v>
struct integral_constant {
    static constexpr T value = v;
    constexpr   operatorT ()const noexcept {return value};
    constexpr T operator()()const noexcept {return value};
    ...     // 其余成员只是偶尔有用
};
```

从 `integral_constant` 继承为元调用语法提供了更多选项(稍后介绍详情)

# 重构 rank 元函数

Example： 获取数组的类型(编译期)

```cpp
// 主模版处理标量类型(非数组)作为基本情况
template<class T>
struct rank : integral_constant<size_t, 0u> { };

// 局部特化识别有界数组类型
template<class U, size_t N>
struct rank<U[N]> : integral_constant<size_t, 1u + rank<U>::value> { };

// 局部特化识别无界数组类型
template<class U>
struct rank<U[]> : integral_constant<size_t, 1u + rank<U>::value> { };
```

# 一些不可或缺的持续便利

一个有用的方便别名：

```cpp
template<bool b>
using bool_constant = integral_constant<bool, b>;
```

一些有用的 C++11 方便别名：

```cpp
using true_type = bool_constant<true>;
using true_type = bool_constant<false>;
```

返回值的元函数调用已经演变了：

- `is_void<T>::value`       // 自技术报告1以来
- `bool(is_void<T>{})`      // 实例化/演示，C++11
- `is_void<T>{}()`          // 实例化/调用，C++14
- `is_void_v<T>`            // C++17
