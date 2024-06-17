---
title: "手把手带你使用 Rust 实现链表"
subtitle:
date: 2024-06-15T20:24:56+08:00
# draft: true
# author:
#   name:
#   link:
#   email:
#   avatar:
description:
keywords:
license:
comment: false
weight: 0
tags:
  - Rust
  - Sysprog
  - Linked List
categories:
  - Rust
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRss: false
summary:
resources:
  - name: featured-image
    src: featured-image.jpg
  - name: featured-image-preview
    src: featured-image-preview.jpg
toc: true
math: true
lightgallery: true
password:
message:
repost:
  enable: true
  url:

# See details front matter: https://fixit.lruihao.cn/documentation/content-management/introduction/#front-matter
---

{{< admonition abstract >}}
其它语言：兄弟，语言学了吗？来写一个链表证明你基本掌握了语法。

Rust 语言: 兄弟，语言精通了吗？来写一个链表证明你已经精通了 Rust！
{{< /admonition >}}

<!--more-->

{{< link href="https://www.bilibili.com/video/BV1eb4y1Q7FA/" content="教学录影" external-icon=true >}}
/
{{< link href="https://rust-unofficial.github.io/too-many-lists/" content="原文地址" external-icon=true >}}
/ 
{{< link href="https://course.rs/too-many-lists/intro.html" content="中文翻译" external-icon=true >}}

## 通过枚举实现 Lisp 风格的链表

```rs
#[derive(Debug)]
enum List<T> {
    Cons(T, Box<List<T>>),
    Nil,
}

fn main() {
    let list: List<i32> = List::Cons(1, Box::new(List::Cons(2, Box::new(List::Nil))));
    println!("{:?}", list);
}
```

{{< admonition >}}
该实作将 **链表节点整体** 视为 **枚举** 进行区分，导致空元素也会占据内存空间
{{< /admonition >}}

符号 `[]` 表示数据存放在 stack 上，`()` 则表示数据存放在 heap 上，上面例子的内存分布为:
```
[List 1, ptr] -> (List 2, ptr) -> (Nil)
```

存在的问题:
1. 元素 1 是分配在 stack 而不是 heap 上
2. 最后的空元素 `Nil` 也需要分配空间

而我们预期的内存分布为:
```
[ptr] -> (List 1, ptr) -> (List 2, ptr) -> (Nil)
```

这样的内存分布更加节省 stack 空间，并且将所有的链表节点都放置在 heap 上，这样在链表拆分和合并时就不需要对头结点进行额外考量和处理，下面是两种内存布局在链表拆分时的对比:
```
// first entry in stack
[List 1, ptr] -> (List 2, ptr) -> (List 3, ptr) -> (Nil)
split off 3:
[List 1, ptr] -> (List 2, ptr) -> (Nil)
[List 3, ptr] -> (Nil)

// first entry in heap
[ptr] -> (List 1, ptr) -> (List 2, ptr) -> (List 3, ptr) -> (Nil)
split off 3:
[ptr] -> (List 1, ptr) -> (List 2, ptr) -> (Nil)
[ptr] -> (List 3, ptr) -> (Nil)
```

显然第一种方式在链表拆分时涉及到链表元素在 stack 和 heap 之间的位置变换，链表合并也类似，请自行思考。

但是这个内存布局并不是最好的，我们想要达到类似 C/C++ 的链表的内存布局:
```
[ptr] -> (List 1, ptr) -> (List 2, ptr) -> (List 3, null)
```

## 实作 C/C++ 风格的链表

```rs
type Link<T> = Option<Box<Node<T>>>;

#[derive(Debug)]
pub struct List<T> {
    head: Link<T>,
}

#[derive(Debug)]
struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

{{< admonition >}}
该实作将 链表节点的 **next 指针部分** 视为 **枚举** 进行区分，这样空元素不会占据内存空间
{{< /admonition >}}

### new

```rs
pub fn new() -> Self {
    Self { head: None }
}
```

### push

```rs
pub fn push(&mut self, elem: T) {
    let next = Box::new(Node {
        elem,
        next: self.head.take(),
    });
    self.head = Some(next);
}
```

### pop

```rs
pub fn pop(&mut self) -> Option<T> {
    self.head.take().map(|node| {
        self.head = node.next;
        node.elem
    })
}
```

注意这里的 `pop` 方法并没有对被弹出的节点 node 进行释放

### drop

```rs
fn drop(&mut self) {
    let mut link = self.head.take();
    while let Some(mut node) = link {
        link = node.next.take();
    }
}
```

将每个节点 node 被指向的指针 (`Box` 指针) 都清除掉，这样 Rust 的所有权机制就会将这些节点 node 占据的内存空间进行清理。

### peek

```rs
pub fn peek(&self) -> Option<&T> {
    self.head.as_ref().map(|node| &node.elem)
}

pub fn peek_mut(&mut self) -> Option<&mut T> {
    self.head.as_mut().map(|node| &mut node.elem)
}
```

## Documentations

这里列举视频中一些概念相关的 documentation 

> 学习的一手资料是官方文档，请务必自主学会阅读规格书之类的资料

### Crate [std](https://doc.rust-lang.org/std/index.html) 

> 可以使用这里提供的搜素栏进行搜索 (BTW 不要浪费时间在 Google 搜寻上！)

- Function [std::mem::replace](https://doc.rust-lang.org/std/mem/fn.replace.html)
- Trait [std::ops::Drop](https://doc.rust-lang.org/std/ops/trait.Drop.html)
- method [std::option::Option::take](https://doc.rust-lang.org/std/option/enum.Option.html#method.take)
- method [std::option::Option::map](https://doc.rust-lang.org/std/option/enum.Option.html#method.map)
- method [std::option::Option::as_ref](https://doc.rust-lang.org/std/option/enum.Option.html#method.as_ref)
- method [std::option::Option::as_mut](https://doc.rust-lang.org/std/option/enum.Option.html#method.as_mut)