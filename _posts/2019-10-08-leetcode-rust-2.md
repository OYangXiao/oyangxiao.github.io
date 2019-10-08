---
layout: post
title: LeetCode Rust (2. add two numbers)
feature-img: "assets/img/pexels/linked-list.jpeg"
thumbnail: "assets/img/pexels/linked-list.jpeg"
image: "assets/img/pexels/linked-list.jpeg" #seo tag
tags: [LeetCode, Rust]
author-id: OYX
excerpt_separator: <!--more-->
---

## 没想到那么快就遇上了传说中的 rust 大坑：链表

第二题：[两数相加](https://leetcode-cn.com/problems/add-two-numbers)

<!--more-->

一直听说 rust 的链表是个坑，因为 rust 的所有权机制导致写起来非常困难，这里 leetcode 给的链表的定义其实就有问题。

```rust
// Definition for singly-linked list.
#[derive(PartialEq, Eq, Clone, Debug)]
pub struct ListNode {
  pub val: i32,
  pub next: Option<Box<ListNode>>
}

impl ListNode {
  #[inline]
  fn new(val: i32) -> Self {
    ListNode {
      next: None,
      val
    }
  }
}
```

这样一个链表使用了 clone 和 eq，那么如果我们创建一个循环链表，爆栈是肯定的。

先不管这些，这个问题考的也不是这个。

## Rust 语法

### mutable 引用

这道题的关键语法在于使用可变引用

```rust
let mut result_link:Option<Box<ListNode>> = None;
let mut result_next = &mut result_link;

// ......

*result_next = Some(Box::new(ListNode::new(val)));
result_next = &mut result_next.as_mut().unwrap().next;
```

上面这段代码首先创建了将要返回的链表的第一个元素，此时重点在于获取该元素的 mutable 引用（`&mut`），之后我们就可以通过`*`操作符将其解引用并赋值。

好久没有写 C 了，对于引用和解引用的使用很不熟练，虽然 rust 的引用和 c 里面不一样，但是以上这段操作可以类比下面这段代码

```c
int a = 1;
int* pa = &a;

*pa = 2;
// 此时a的值被修改为2
```

## 题解

（不要犯错就行了）

## 代码

```rust
impl Solution {
  pub fn add_two_numbers(
    l1: Option<Box<ListNode>>,
    l2: Option<Box<ListNode>>,
  ) -> Option<Box<ListNode>> {
    let mut l1_next = l1;
    let mut l2_next = l2;
    let mut result_link:Option<Box<ListNode>> = None;
    let mut result_next = &mut result_link;
    let mut carry = false;

    while carry || l1_next.is_some() ||l2_next.is_some()  {
      let val1 = match l1_next {
        Some(node) => {
          l1_next = node.next;
          node.val
        }
        None => 0,
      };
      let val2 = match l2_next {
        Some(node) => {
          l2_next = node.next;
          node.val
        }
        None => 0,
      };

      let mut val = val1 + val2 + if carry { 1 } else { 0 };
      carry = false;

      if val > 9 {
        val = val - 10;
        carry = true;
      }

      *result_next = Some(Box::new(ListNode::new(val)));
      result_next = &mut result_next.as_mut().unwrap().next;
    }
    result_link
  }
}
```

## 成绩

很悲惨，前几天提交的时候运行时间还是4ms，2MB，怎么过完十一变成了8ms了，过段时间研究研究时间更短的解法。
