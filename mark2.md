---
title: 现代模版元编程
tags: ["template"]
categories: [Cppcon 2014 - Walter E. Brown]
description: '深挖 C++ 模版知识'
date: 2024-09-18 09:58:21
cover: https://s2.loli.net/2024/09/16/NU7CT6YbWmAcDfu.png
---

# 将继承和专业化结合起来

Example 1： 给一个类型，它会是 `void` 类型吗

```cpp
// 非空类型的主模版
template<class T>
struct is_void : false_type { }; // 主模版假设你给了一个不是 void 的类型

// 特化识别四种空白类型中的每一种
template<>
struct is_void<void> : true_type { };

template<>
struct is_void<void const> : true_type { };
...
```

Example 2：给出两种类型，他们是相同的吗

```cpp
// 主模版：不相同的情况
template<class T, class U>
struct is_same : false_type { };

// 特化识别相同类型的情况
template<class T>
struct is_same<T, T> : true_type { };
```

# 转发/去除其他的元函数

Example：给一个类型，它是 `void` 吗

```cpp
template<class T>
using is_void = is_same<remove_cv_t<t>, void>;      // remove_cv_t 去除 cv 限定

// 其中的 remove_cv_t 也很简单
template<class T>
using remove_cv = remove_volatile<remove_const_t<T>>;

// C++14 中有 _t 的版本
template<class T>
using remove_cv_t = typename remove_cv<T>::type;
```

上述例子中的 `remove_cv` 前面加一个 `typename` 关键字的意义是，告诉编译器，`remove_cv<T>::type` 是一个类型，否则编译器不能正确解析这一点(待决名)。

# 在元函数中使用参数包

Example：将 `is_same` 归纳为 `is_one_of`

```cpp
// 主模版：T 是否与 P0toN 中任一类型相同
template<class T, class...P0toN>
struct is_one_of;       // 只声明接口

// 给我一个类型，和一个任意长的类型列表，我想知道我的类型是不是其中之一

// 特化 #1：这个类型列表(参数包)是空的
template<class T>
struct is_one_of<T> : false_type { };

// 特化 #2：T 与这个类型列表的第一个类型匹配
template<class T, class...P1toN>
struct is_one_of<T, T, P1toN...> : true_type { };

// 特化 #3：T 与这个类型列表的之后的类型匹配，递归展开
template<class T, class P0, class...P1toN>
struct is_one_of<T, P0, P1toN...> : is_one_of<T, P1toN...> { };
```

# 重定向 is_void

Example：给一个类型，它会是 `void` 类型吗

```cpp
template<class T>
using is_void = is_one_of<T, void, void const, void volatile, void const volatile>;
```

# 未计算的操作数

回想一下，`sizeof`，`typeid`，`decltype`，`noexcept`的操作数未被计算过，即使在编译期也是如此，他们都是不求值表达式，这意味着(在上下文中)，不会为这些操作数表达式生成代码，所以没有运行成本。意味着我们只需要一个声明，而不是一个定义，就可以在这些上下文中使用一个(函数或对象的)名称。

不求值的函数调用可以有效的将一种类型映射到另一种类型：

```cpp
decltype(foo(daclval<T>))   // 给出 foo 的返回值类型，如果他被调用
```

未评估的调用 `std::declval<T>()` 被声明为给出类型为 `T` 的 rvalue 结果。(`std::declval<T&>()` 给出 lvalue)。

# 对拷贝可分配性进行测试

```cpp
template<class T>
struct is_copy_assignable {
private:
    template<class U, class = decltype(declval<U&>() = declval<U const&>())>
    static true_type try_assignment(U&&);   

    static false_type try_assignment(...);
public:
    using type = decltype(try_assignment(declval<T>()));
};
```

# 旧技术

在 C++11 引入 `decltype` 之前，`sizeof` 被 `(ab)` 用来提供未评估的元编程上下文：在阅读早期(C++98/C++03)模板元编程代码时，记住这一点很有用。

详情如下：
1. 重载的返回类型被设计成具有不同的大小，例如，`typedef char(&yes)[1];` 和 `typedefchar(&no)[2];`
2. 在以前，在为评估的上下文中调用重载的 `sizeof(try_assignment(...))`
3. 为了确定选择哪个重载：`typedf bool_constant<sizeof(...) == sizeof(yes)> type;` 

# 建议的新类型 void_t

简单的具体来说：

```cpp
template<calss...>
using void_t = void;
```

这个特定的 `void_t` 模板别名曾经引发了一个与类模板部分特化相关的 Bug。这个问题源于在某些编译器（尤其是早期的编译器版本）中，`void_t` 有时无法正确地在模板推导中触发 SFINAE，导致程序尝试实例化非法的模板，进而触发编译错误。这与模板参数推导的顺序和编译器如何处理类型的替代失败有关。

无论哪种情况，对 `void` 的另一种拼写对我们有什么帮助呢?其实不一定是 `void`，我们只是需要一种可以预测的东西，选择 `void` 只是因为他写起来方便一点。

# void_t 的效用

充当元函数调用，将任何格式良好的类型映射到(可预测!)的类型空格中：最初是一个实现细节，同时提出了 `common_type` 和 `iterator_traits` 的 SFINAE 友好版本。

Example：检测名为 `T::type` 的类型的成员存在/不存在

```cpp
// 主模版
template<class, class = void>
struct has_type_member : false_type { };

// 模版特化
template<class T>
struct has_type_member<T, void_t<typename T::type>> : true_type { };
```

# 重新审阅 is_copy_assignable

```cpp
// 用于有效副本指派结果类型的帮助别名:
template<class T>
using = copy_assignment_t = decltype(declval<T&>() = declval<T const&>());

//主模板处理所有非复制分配类型:
template<class T, class = void>
struct is_copy_assignable : false_type { };

//特化仅识别和验证拷贝可分配类型:
template<class T>
struct is_copy_assignable<T, void_t<copy_assognment_t<T>>> : is_same<copy_assignment_t<T>, T&> { };
```

1. `copy_assignment_t` 别名模板：

```cpp
template<class T>
using copy_assignment_t = decltype(declval<T&>() = declval<T const&>());
```

这行代码通过 `decltype` 推导出 `T& = T const&` 表达式的类型。也就是说，它尝试检查 `T` 是否可以通过 `const T&` 进行复制赋值。如果 `T` 支持这种赋值操作，则 `copy_assignment_t<T>` 会得到 `T&` 作为类型；否则，会导致 SFINAE 失败。

- `declval<T&>()`：用于生成一个左值引用 `T&`，用于模拟左值赋值操作。
- `declval<T const&>()`：用于生成一个 `const T&`，用于模拟一个右值作为赋值来源。
- `decltype(...)`：获取赋值表达式的结果类型。

2. 主模板 `is_copy_assignable`：

```cpp
template<class T, class = void>
struct is_copy_assignable : false_type { };
```

这个模板的主模板假设 所有类型 `T` 都不是可复制赋值类型，默认继承自 `false_type`。也就是说，如果没有提供其他特化版本，`is_copy_assignable<T>` 将返回 `false。`

3. 特化版本 `is_copy_assignable`：

```cpp
template<class T>
struct is_copy_assignable<T, void_t<copy_assignment_t<T>>> : is_same<copy_assignment_t<T>, T&> { };
```

这是针对可复制赋值类型的特化版本。它使用了 `void_t` 和 `copy_assignment_t<T>` 来进行 SFINAE 检查。

- `void_t<copy_assignment_t<T>>`：如果 `copy_assignment_t<T>` 有效（即 `T` 支持复制赋值），那么 `void_t` 会生成 `void`，因此选择这个特化版本。
- `is_same<copy_assignment_t<T>, T&>`：检查 `copy_assignment_t<T>` 是否为 `T&`。也就是说，它检查赋值操作的结果是否是左值引用 `T&`。如果是，则类型` T` 支持复制赋值，`is_copy_assignable<T>` 将返回 `true`。

# 技术和工具概述

- 元函数成员类型、静态 `const` 数据成员和 `constexpr`` 成员函数来表示中间和最终元函数结果。
- 元函数通过调用(可能为递归调用)、继承和别名以实现因子共性。
- 元函数参数模式匹配的模板专门化(完全和部分)
- SFINAE可直接解决重载问题。
- 未估值的操作数，如映射类型的函数调用。
- 参数包为列表(通常是类型列表)
- 在 `<type_traits>`、`<iterator>`、`<numeric limits>` 等中和经典的std::元函数。