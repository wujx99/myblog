---
title: modern template metaprogramming
date: 2023-11-04 22:52:10
tags:
- cpp
- cppcon
---

ppt见[modern-template-metaprogramming](/引用资源/cppcon/)

<!-- more -->

>当我们使用模板元编程的时候要记住：Keep in mind that run-time==compile-time,so can’t rely on:☒mutability,☒virtual functions,	☒ other RTTI,etc.


# 把工作转移到compile time

## metafunction
```cpp
template<int N>		//	template param used as the metafctn param
struct abs{	
    static_assert(N != INT_MIN);			    // cpp17-styleguard
	static constexpr auto value = (N < 0) ?	–N :N;	 // “return”
};	

//如下的调用
int	const n	= … ;		//	could instead declare as constexpr
…abs<n>::value…			//	instantiation yields a compile-time constant
```

与cpp的constexpr function有什么区别呢？模板`abs`是一个`struct`！！！可以在内部进行声明等！

## 例子

```cpp compile-time-recursion.cpp
template< unsigned M, unsigned N > struct gcd { // per Euclid 
    static constexpr auto value = gcd<N, M%N>::value; };

//特例化来创建递归基
template< unsigned M > struct gcd { 
    static_assert( M != 0 ); // gcd(0, 0) is undefined, so disallow 
    static constexpr auto value = M; 
}
```

## 定义自己的sizeof（输入时类型）
```cpp my-sizeof.cpp
// primary template handles scalar (non-array) types as base case: 
template< class T > 
    struct rank { static constexpr size_t value = 0u; }; 
// partial specialization recognizes any array type: 
template< class U, size_t N > 
    struct rank< U[N] > { static constexpr size_t value = 1u + rank<U>::value; }; 



//Usage: 
 using array_t = int [10] [20] [30]; 
 … rank<array_t>::value … // yields 3u (at compile-time)

```

## metafunction可以返回类型
去掉顶层const
```cpp remove-top-const.cpp
// primary template handles types that are not const-qualified: 
template< class T > 
struct remove_const { using type = T; }; // identity 

// partial specialization recognizes const-qualified types: 
template< class U > 
struct remove_const< U const > { using type = U; };



//Usages (call syntax): 
remove_const<T>::type t;   // ensure t has a mutable type 
remove_const_t<T> t;       // cpp14 equivalent; more later

```

## lib metafunction convention#1

1. 返回类型的meta function必须要用别名`type`

**一个例子**
```cpp
template< class T > 
struct type_is { using type = T; }; 
//Convenient to apply the convention via public inheritance:  

// primary template handles types that are not volatile-qualified: 
template< class T > 
struct remove_volatile : type_is< T > { }; // identity 
// partial specialization recognizes volatile-qualified types: 
template< class U > 
struct remove_volatile< U volatile > : type_is< U > { };
```

## compile-time decision-making

```cpp 
template< bool p, class T, class F > 
struct IF : type_is< … > { }; // p ? T : F
```
怎么根据p的值决定类型的是T或者F？还是老办法，默认加模板特例化
```cpp
// primary template assumes the bool value is true: 
template< bool, class T, class >    // needn’t name unused param’s 
struct IF : type_is< T > { };       
// partial specialization recognizes a false value: 
template< class T, class F > 
struct IF<false, T, F> : type_is< F > { }; 
```

> 在cpp11中`IF`别命名为`conditional`


上面的一个变种，要么true返回类型，false什么都不返回

```cpp
// primary template assumes the bool value is true: 
template< bool, class T = void > // default is useful, not essential 
struct enable_if : type_is< T > { }; 
// partial specialization recognizes a false value, computing nothing: 
template< class T > 
struct enable_if<false, T> { }; // no member named type! 
```

调用特例化的metafunction会返回error吗？不会！会有一个与模板无关的机制SFINAE:SFINAE: Substitution Failure Is Not An Error. (Also sometimes termed explicit overload set management.)


模板实例化过程

1. Obtain (or figure out) the template arguments.(直接，通过参数推断，默认模板参数三个步骤)
2. Replace each template parameter, throughout the template, by its corresponding template argument.


如果上述过程失败了（这不是一个错误），就会被默认的**丢弃掉**！我们要借用SFINAE的特性


## SFINAE in use

Example: want one algorithm f taking integral types T, and overload it with a second f taking floating-point types T.

对于下面的两个模板，我们只有一个实例化.
```cpp
template< class T > 
enable_if_t< is_integral<T>::value, maxint_t > //返回类型,如果第一个参数是true，那么结果就是第二个参数
    f ( T val ) { … };

template< class T > 
enable_if_t< is_floating_point<T>::value, long double > //返回类型
    f ( T val ) { … };
```

如果两者都不匹配则会出现错误！！


## concepts

Its “constraints” metaprogramming feature seems likely to reduce or obviate many current uses for **SFINAE**, etc.

比如下面的SFINAE例子对比：
```cpp
template< class T > enable_if_t< is_integral<T>::value, maxint_t > // SFINAE 
    f ( T val ) { … };  
template< Integral T > // constrained template (short form) 
    maxint_t f ( T val ) { … };
```


## lib metafunction convention#2
2. 这个约定用于的meta function with value result
- A static constexpr member, value, giving its result, and …
- A few convenience member types and constexpr functions.

下面的就是cpp标准 value-returning metafunction中的例子
```cpp
template< class T, T v > 
struct integral_constant { 
    static constexpr T value = v; 
    constexpr operator T ( ) const noexcept { return value; } 
    constexpr T operator ( ) ( ) const noexcept { return value; } 
    … // remaining members are only occasionally useful 
    };
```

我们回看我们的rank metafuction，我们还加入了无界数组的rank
```cpp
// primary template handles scalar (non-array) types as base case: 
template< class T > 
struct rank : integral_constant< size_t, 0u > { }; 
// partial specialization recognizes bounded array types: 
template< class U, size_t N > 
struct rank< U[N] > 
: integral_constant< size_t, 1u + rank<U>::value > { }; 
// partial specialization recognizes unbounded array types: 
template< class U > 
struct rank< U[ ] > 
: integral_constant< size_t, 1u + rank<U>::value > { };
```

一些`integral_constant`方便之处：
{% asset_img img01.png %}

> 什么是变量模板？在上图中有什么用？

## using inheritance + specialization together

Example 1: given a type, is it a void type?

```cpp
// primary template handles non-void types: 
template< class T > struct is_void : false_type { }; 
// four specializations, one to recognize each of the four void types:
 template< > struct is_void<void> : true_type { }; 
 template< > struct is_void<void const> : true_type { }; 
```
> 会使用不同的方式判断是否为void，注意总结

Example 2: given two types, are they one and the same? 
```cpp
// primary template handles dis?nct types: 
template< class T, class U > struct is_same : false_type { }; 
// partial specialization recognizes identical types: 
template< class T > struct is_same<T, T> : true_type { };
```

## Aliasing == delegation + binding

Example: given a type, is it a void type?
```cpp
template< class T > 
using remove_cv = remove_volatile< remove_const_t<T> >; 
template< class T > 
using remove_cv_t = typename remove_cv<T>::type;

//如下使用上面的定义来判断是否为
template< class T > 
using is_void = is_same< remove_cv_t<T> , void >;

```
> 太牛了！！

## Using a parameter pack in a metafunction

Example: generalize is_same into is_one_of:


```cpp
// primary template: is T the same as one of the types P0toN… ? 
template< class T, class... P0toN > 
struct is_one_of; // declare the interface only 
// base #1: specialization recognizes empty list of types: 
template< class T > 
struct is_one_of : false_type { }; 
// base #2: specialization recognizes match at head of list of types: 
template< class T, class... P1toN > 
struct is_one_of : true_type { }; 
// specialization recognizes mismatch at head of list of types: 
template< class T, class P0, class... P1toN > 
struct is_one_of : is_one_of { }; // go inspect list’s tail
```
> 用了类似递归的方式实现的


**再次重写**`is_void`!!!
Example: given a type, is it a void type?
```cpp
 template< class T > 
 using is_void = is_one_of< T , void , 
            void const , void volatile , void const volatile >;
```

## Unevaluated operands

`sizeof`,`alignof`,`typeid`,`decltype`,`noexcept`都是unevaluated的，所以只需要声明，而不需要定义。

```cpp
decltype( foo( declval( ) ) ) // declval is in <utility>
            // gives foo’s return type, were it called with a T rvalue
```

下面是一个例子
{% asset_img img02.png %}

decltype以前的通常使用sizeof在下文，对比一下

{% asset_img img03.png %}

## Proposed new type trait void_t

下面的是有问题的
```cpp
template< class ... > 
using void_t = void;
```

## Utility of void_t

Acts as a metafunction call that maps any well-formed type(s) into the (predictable!) type void！

Example: detect the presence/absence of a valid type member named T::type (per the metafunction convention): 

```cpp
// primary template: 
template< class, class = void > 
struct has_type_member : false_type { }; 
// partial specialization: 
template< class T > 
struct has_type_member< T, void_t< typename T::type > > 
: true_type { };
```

上面的很难看懂把，看看下面的解释
{% asset_img img04.png %}

然后我们就可以回过头再看我们的is_copy_assignable

```cpp
// recall this helper alias for the result type of a valid copy assignment: 
template< class T > 
using copy_assign_t = decltype( declval<T&>( ) 
= declval< T const& >( ) ); 
// primary template handles all non-copy-assignable types: 
template< class T, class = void > // default argument is essential 
struct is_copy_assignable : false_type { }; 
// specialization recognizes and validates only copy-assignable types: 
template< class T > 
struct is_copy_assignable< T, void_t< copy_assign_t > > 
: true_type { };
```

> Want is_move_assignable? Change T const& ➞ T&&.


## 总结

还有一些内容应为时间没有涉及到！
{% asset_img img05.png %}