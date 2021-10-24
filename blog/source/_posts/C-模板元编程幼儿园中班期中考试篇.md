---
title: C++模板元编程幼儿园中班期中考试篇
date: 2021-10-24 23:10:38
tags:
---
## 背景

[https://mp.weixin.qq.com/s/E3-MUoN3LmtL9cp1n14Ydw](https://mp.weixin.qq.com/s/E3-MUoN3LmtL9cp1n14Ydw)

本文素材来自于上面这个链接，一个怪人用88万行switch case完成了下面这个题目。 

- 给出一个不多于5位的正整数，要求：
1 求出它是几位数；
2 分别输出每一位数；
3 按逆序输出各位数宇，例如原数为 5631，应输出为 1365。
    
    

 在原文调侃下，实际上实现了这个题目（不考虑代码体积）的性能理论最佳的实现，不需要任何计算，直接switch case出结果。

看了大佬的实现之后，我发现其实这个代码完全可以用c++ 的metaprogramming魔法来写，就是一个预计算的代码生成。所以以我现在c++元编程的幼儿园中班水平，花了亿点点时间写了这个实现。

## 实现

最终代码。[https://github.com/mOnkD404/testcode/blob/master/variadic.cc](https://github.com/mOnkD404/testcode/blob/master/variadic.cc)

 核心逻辑总共不到100行，下面开始实现。

1. 首先需要做到类似switch case  的代码逻辑，不能有连续的if else，要能通过偏移直接找到代码地址，也就是function dispatch table的实现方式。  然后如果想要和switch case一样没有运行时额外开销，就不能用static const变量实现，因为static是第一次运行时跑初始化逻辑。在这里使用constexpr 构造函数的特性实现 (constexpr constructor)。这部分都是标准 c++ 写法，基本没啥新鲜的。

```cpp
template <size_t N>
struct PrintTable<N> {
  std::array<void (*)(), N> table_{};

  constexpr PrintTable() {}
  void PrintAt(int index) const { table_[index](); }
};

void PrintInt(int c) {
  constexpr int size = 1000;
  if (c >= size)
    return;

  using Table = PrintTable<size>;
  constexpr Table tb;
  tb.PrintAt(c);
}
```

1.  接下来需要填充table的初始化逻辑，我们希望能用模版的变参展开特性，让编译器在编译阶段帮我们生成这个从1 到 N 的 table，那么模板的参数就不能是size_t,  这里需要用到 c++  的  index sequence 这个特性。调整之后代码长这样 。 其中前两行是模板的空实现，目的是为了使用 index sequence提供偏特化。

```cpp
template <typename T>
struct PrintTable;
template <size_t... N>
struct PrintTable<std::index_sequence<N...>> {
  std::array<void (*)(), sizeof...(N)> table_{PrintTraits<N>::Print...};

  constexpr PrintTable() {}
  void PrintAt(int index) const { table_[index](); }
};

void PrintInt(int c) {
  constexpr int size = 1000;
  if (c >= size)
    return;

  using Table = PrintTable<std::make_index_sequence<size>>;
  constexpr Table tb;
  tb.PrintAt(c);
}
```

1.  接下来实现 PrintTraits 。PrintTraits 要实现题目的要求，输出几位数，每位数是什么，然后倒序输出。 对每个要求的输出实现一个traits模板。
    1. 首先我们实现一个帮助模板，用于记录一个已经拆分完的数字。 显然，他自然就有了输出序列，和计算序列长度的方法。这里用了模板变参展开的日常生活小技巧。
    
    ```cpp
    template <char... Is>
    struct DigitsVector {
      using Type = DigitsVector<Is...>;
      enum { DigitsCount = sizeof...(Is) };
      static void PrintDigitsString() { ((std::cout << (int)Is << " "), ...); }
      static void PrintDigitsCount() { std::cout << DigitsCount << " "; }
    };
    ```
    
    b. 那么最关键的一步就是要把模板的输入数字拆分。这部分我一开始的实现是比较挫的，在群里跟一些c++魔法爱好者学习之后，才写出了比价符合c++魔法部价值观的实现。
    
    ```cpp
    template <int IntVal, typename = DigitsVector<>>
    struct DigitTraitsHelper;
    
    template <int IntVal, char... Is>
    struct DigitTraitsHelper<IntVal, DigitsVector<Is...>>
        : DigitTraitsHelper<IntVal / 10, DigitsVector<IntVal % 10, Is...>> {};
    
    template <int... Is>
    struct DigitTraitsHelper<0, DigitsVector<Is...>> : DigitsVector<Is...> {};
    ```
    
     最重要的需要养成的习惯是，c++ 元编程是面向编译器的。最最常用的技巧是利用编译器对类型的推导和解析能力，来实现常规代码中的函数调用和递归或循环，来实现计算功能，比如这里，通过每次继承就在DigitsVector中塞入一个参数的方式，将一个位数的信息传入DigitsVector模板参数中，最后在第一个参数为0时，停止继续继承自己，类似递归逻辑的结束点。
    
     c.  现在序列已经生成，接下来需要实现类似“个位数：1，十位数：8”这种输出的逻辑。这部分经过一些思考之后，我们发现可以构造一个静态的prefix table，然后还是用index sequence的方式，配合变参模板输出即可。这里面用了 static_assert 和 std::extent  来断言防止位数超过支持的宽度。并且继续使用了变参展开的生活小技巧。~~小技巧小技巧，一天不学受不了~~
    
    ```cpp
    template <typename T, typename N>
    struct PrintDigitsNameHelper;
    
    template <char... Is, size_t... N>
    struct PrintDigitsNameHelper<DigitsVector<Is...>, std::index_sequence<N...>> {
      constexpr const static char* NameTable[] = {
          "ones digit", "tens digit", "hundreds digit", "thousands digit",
          "ten thousands digit"};
    
      constexpr static size_t C = sizeof...(N);
      static_assert(C <= std::extent<decltype(NameTable)>::value, "overflow");
      static void Print() {
        ((std::cout << NameTable[C - N - 1] << " : " << (int)Is << " "), ...);
      }
    };
    ```
    
    d. 最后是一个反序的模板实现，有了上文的铺垫之后，到这里已经十分的straight forward。就是利用类型推倒和继承机制，把一个DigitsVector的模板参数反序。实际就是在利用编译器的type推倒机制实现递归，最后递归结束点是原DigitsVector 的参数为空。注意这里很容易想到将变参模板的参数用普通函数的方式递归之后反序输出，不够魔幻，会被魔法部警告。
    
    ```cpp
    template <typename T, char... Rs>
    struct DigitVectorReverseHelper;
    
    template <int IntVal, char... Is, char... Rs>
    struct DigitVectorReverseHelper<DigitsVector<IntVal, Is...>, Rs...>
        : DigitVectorReverseHelper<DigitsVector<Is...>, IntVal, Rs...> {};
    
    template <char... Rs>
    struct DigitVectorReverseHelper<DigitsVector<>, Rs...> : DigitsVector<Rs...> {};
    ```
    
    e. 最后来实现打印逻辑，就是把上面几个输出放一起。
    
    ```cpp
    template <int N>
    struct PrintTraits {
      using Helper = typename DigitTraitsHelper<N>::Type;
      using ReverseHelper = typename DigitVectorReverseHelper<
          typename DigitTraitsHelper<N>::Type>::Type;
      using PrintDigitsName =
          PrintDigitsNameHelper<Helper,
                                std::make_index_sequence<Helper::DigitsCount>>;
    
      static void Print() {
        std::cout << " total: ";
        Helper::PrintDigitsCount();
        PrintDigitsName::Print();
        std::cout << " in order: ";
        Helper::PrintDigitsString();
        std::cout << " reverse: ";
        ReverseHelper::PrintDigitsString();
        std::cout << std::endl;
      }
    };
    ```
    

最后的最后补全一些判断逻辑，是见证魔法的时刻。

```cpp
#include <array>
#include <cmath>
#include <iostream>
#include <type_traits>

template <char... Is>
struct DigitsVector {
  using Type = DigitsVector<Is...>;
  enum { DigitsCount = sizeof...(Is) };
  static void PrintDigitsString() { ((std::cout << (int)Is << " "), ...); }
  static void PrintDigitsCount() { std::cout << DigitsCount << " "; }
};

template <typename T, typename N>
struct PrintDigitsNameHelper;

template <char... Is, size_t... N>
struct PrintDigitsNameHelper<DigitsVector<Is...>, std::index_sequence<N...>> {
  constexpr const static char* NameTable[] = {
      "ones digit", "tens digit", "hundreds digit", "thousands digit",
      "ten thousands digit"};

  constexpr static size_t C = sizeof...(N);
  static_assert(C <= std::extent<decltype(NameTable)>::value, "overflow");
  static void Print() {
    ((std::cout << NameTable[C - N - 1] << " : " << (int)Is << " "), ...);
  }
};

template <int IntVal, typename = DigitsVector<>>
struct DigitTraitsHelper;

template <int IntVal, char... Is>
struct DigitTraitsHelper<IntVal, DigitsVector<Is...>>
    : DigitTraitsHelper<IntVal / 10, DigitsVector<IntVal % 10, Is...>> {};

template <int... Is>
struct DigitTraitsHelper<0, DigitsVector<Is...>> : DigitsVector<Is...> {};

template <typename T, char... Rs>
struct DigitVectorReverseHelper;

template <int IntVal, char... Is, char... Rs>
struct DigitVectorReverseHelper<DigitsVector<IntVal, Is...>, Rs...>
    : DigitVectorReverseHelper<DigitsVector<Is...>, IntVal, Rs...> {};

template <char... Rs>
struct DigitVectorReverseHelper<DigitsVector<>, Rs...> : DigitsVector<Rs...> {};

template <int N>
struct PrintTraits {
  using Helper = typename DigitTraitsHelper<N>::Type;
  using ReverseHelper = typename DigitVectorReverseHelper<
      typename DigitTraitsHelper<N>::Type>::Type;
  using PrintDigitsName =
      PrintDigitsNameHelper<Helper,
                            std::make_index_sequence<Helper::DigitsCount>>;

  static void Print() {
    std::cout << " total: ";
    Helper::PrintDigitsCount();
    PrintDigitsName::Print();
    std::cout << " in order: ";
    Helper::PrintDigitsString();
    std::cout << " reverse: ";
    ReverseHelper::PrintDigitsString();
    std::cout << std::endl;
  }
};

template <>
struct PrintTraits<0> {
  static void Print() {
    std::cout << " total: 1 ones digit: 0 in order: 0 reverse: 0 " << std::endl;
  }
};

template <typename T>
struct PrintTable;
template <size_t... N>
struct PrintTable<std::index_sequence<N...>> {
  std::array<void (*)(), sizeof...(N)> table_{PrintTraits<N>::Print...};

  constexpr PrintTable() {}
  void PrintAt(int index) const { table_[index](); }
};

void PrintInt(int c) {
  constexpr int size = 100000;
  if (c >= size)
    return;

  using Table = PrintTable<std::make_index_sequence<size>>;
  constexpr Table tb;
  tb.PrintAt(c);
}

int main(int argc, char* argv[]) {
  for (std::string line; std::getline(std::cin, line);) {
    PrintInt(atoi(line.c_str()));
  }
  return 0;
}
```

这文件编译出来有亿点大，135MB，十分感人。
