---
layout: post
title: LeetCode Rust (1)
tags: [LeetCode, Rust]
author-id: OYX
excerpt_separator: <!--more-->
---

# 开始用 Rust 刷 LeetCode

学了 rust 总不能天天写 web server，毕竟同样的东西写来写去意义不大，但是又缺少一个合适的练手项目。正好发现 leetcode，那么就开始刷题吧。

第一题：[两数之和](https://leetcode-cn.com/problems/two-sum)

<!--more-->

总结一下遇到的相关的问题

## Rust语法

### for的基本用法

常见的教程上都是
{% highlight js %}
for item in iterator
{% endhighlight %}
即使用 for 对一个迭代器（iterator）做迭代。

rust中for不可以直接迭代数组，只能迭代iterator，常见的迭代器来源有：

- range（比如：1..11，字符串的各个字符的range）
- 从数组生成的 iterator（比如：vec.iter()）

可以关联到常用的迭代器的操作符（map、rev...）这些。

回到两数之和这个问题，我们需要取出某个数在数组中的下标，则简单的对数组迭代无法实现，iterator 的 enumerate 方法可以返回被迭代元素的 index 和元素本身合成的一个元组，所以这题的语法关键问题就是
{% highlight js %}
for (index, value) in nums.iter().enumerate()
{% endhighlight %}

### usize 和 as转换类型

usize与CPU和操作系统相关，使用usize总是可以表示一个指向内存中的具体地址，32为系统中这个值是4-byte，64位中是8-byte。
enumerate中获得的index都是usize类型的，由于题目只接受i32类型的返回结果，需要显式使用as将其转换为i32类型。

## 题解

本题的技巧主要在于减少循环的次数，如果简单的先将所有数字放入hashmap中，然后遍历所有数字，则需要O(2n)的时间，如果将数组中数字放入hashmap的时候就与hashmap中已经存在的数字作比较（hashmap中已经存在的数字都是向前匹配失败的数字），那么只需要一次循环就完成检索，时间复杂度是O(n)
