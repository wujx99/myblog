---
title: classic STL
date: 2023-11-06 14:10:55
tags:
- cppcon
- cpp
---

ppt见[stl](/引用资源/cppcon/)

<!-- more -->

# Rationale

{% asset_img img01.png %}

# History and design overview

## 最初的设计准则

{% asset_img img02.png %}

## Key Principles

{% asset_img img03.png %}

## Complexity and interfaces

{% asset_img img04.png %}

## Containers Overview

{% asset_img img05.png %}
{% asset_img img06.png %}

## Itertors Overview

{% asset_img img07.png %}
{% asset_img img08.png %}

## Algorithms Overview

{% asset_img img09.png %}


# Iterators

跟着ppt的步骤回答下面的问题，也回顾了整个的设计！
{% asset_img img10.png %}

80年代c时代只有指针，我们可以在常量时间做数组指针操作，线性时间的双链表指针操作（指针操作更少，不能有p++），单链表指针操作。

## Multi-pass and Single-Pass Iteration

上面的三个数据结构支持multi-pass iteration，但是如read/write to sockets，file streams就是single-pass iteration；如`File* p = fopen(mame, "rd")`的`p`。

## iterator Catagories

{% asset_img img11.png %}

## Iterator Ranges

N个元素一般会有N+1个position，最后一个不能dereferenceable。

iterator range一般是前闭后开`[first, last）`

> 怎么实现last呢?如果是连续存储的容器（vector等）就只需要last=尾后。如果是node-based 容器（list等）就设置哨兵node。


## five iterator interface bottom to top

主要介绍了上面图中五种iterator的接口。

注意细节！！！比如`*p`,`p==q`的含义的不同

## Iterator Adaptor




# Containers

介绍了一些基本的容器和容器适配器。

# Algorithms

算法的内容有很多，就选取了一些合适的讲解一下。

```cpp sort.cpp
template<class RandomIter, class Compare>
void
sort(RandomIter first, RandomIter last, Compare comp);
```

```cpp lower_bound.cpp
template<class ForwardIter, class T>
ForwardIt
lower_bound(ForwardIter first, ForwardIter last, const T& value);
```


```cpp remove_copy_if.cpp
template<class InputIter, class OutputIter, class UnaryPredicate>
OutputIt
remove_copy_if(InputIter first, InputIter last, OutputIter dest, UnaryPredicate pred)
{
    for (; first != last; ++first)
    {
        if (!pred(*first))
        {
            *dest++ = *first;
        }
    }
    return dest;
}
```

