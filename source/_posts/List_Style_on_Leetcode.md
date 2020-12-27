---
title: 刷题总结-链表类型题目
date: 2020-09-06 19:23:00
tags:
---
<!-- toc -->

--根据[LeetCode的题目分类清单](https://blog.csdn.net/CowBoySoBusy/article/details/82559651)刷完链表部分题目，总结如下

## 1 题目类型分类

- 插入
- 排序
- 去重
- 反转
- 合并
  - 合并多个链表
- 拷贝
  - 返回链表拷贝，链表可能含有指针
- 子串
  - 链表中查找特定子串
- 移动
  - 前移后移
- 拆分
  - 按照一定逻辑提取节点
- 交换
- 检定
  - 检查链表是否符合条件

## 2 主要操作手段

### 2.1 定义链表

```java
public class ListNode {
  int val;
  ListNode next;

  ListNode() {
  }

  ListNode(int val) {
    this.val = val;
  }

  ListNode(int val, ListNode next) {
    this.val = val;
    this.next = next;
  }
}
```

### 2.2 遍历

```java
public void travel(ListNode head, Consumer<ListNode> consumer) {
  ListNode iter = head;
  while (iter != null) {
    consumer.accept(iter);
    iter = iter.next;
  }
}
```

 遍历没啥说的

### 2.3 插入

```java
public ListNode insert(ListNode before, ListNode beInserted) {
  ListNode next = before.next;
  before.next = beInserted;
  beInserted.next = next;
  return before;
}
```

逻辑如下：

- 暂存当前节点的下一个节点
- 赋值当前节点的下一个节点为待插入节点
- 赋值待插入节点的下一个节点为缓存的节点

### 2.4 删除

```java
public ListNode remove(ListNode before) {
  ListNode iter = before;
  if (iter.next != null) {
    ListNode toBeRevmoed = iter.next;
    iter.next = toBeRevmoed.next;
    return toBeRevmoed;
  }
  return null;
}
```

逻辑如下：

- 设置当前节点的下一个节点为下下个节点

### 2.5 反转

```java
public ListNode reverse(ListNode head) {
  ListNode iter = head, prev = null;
  while (iter != null) {
    ListNode next = iter.next;
    iter.next = prev;
    prev = iter;
    iter = next;
  }
  iter.next = prev;
  return iter;
}
```

逻辑如下：

- 暂存前一个节点/当前节点
- 暂存当前节点的下一个节点
- 设置当前节点的下一个节点为前一个节点
- 更新前一个节点至当前节点
- 更新当前节点至缓存的下一个节点

### 2.6 查半

```java
public ListNode findMiddle(ListNode head) {
  ListNode iter = head;
  ListNode mid = head;
  while (iter != null & iter.next != null) {
    iter = iter.next.next;
    if (iter == null) {
      break;
    }
    mid = mid.next;
  }
  return mid;
}
```

逻辑如下：

- 设置两个指针一快一慢
- 快指针以慢指针的2倍速前移

## 3 常用算法

暂时只用到一个归并排序

### 3.1 归并排序

归并排序基于分治手段，总的说有三个步骤：

- 分解：将含n个元素的列表分成两个含n/2个元素的子列
- 解决：对子列进行排序
- 合并：合并两个子列

实现如下：

```java
public ListNode sortList(ListNode head) {
  // check
  if (head == null || head.next == null) {
    return head;
  }

  ListNode prev = null, mid = head, iter = head;
  while (iter != null && iter.next != null) {
    prev = mid;
    mid = mid.next;
    iter = iter.next.next;
  }

  prev.next = null;

  ListNode l1 = sortList(head);
  ListNode l2 = sortList(mid);

  return merge(l1, l2);
}

ListNode merge(ListNode l1, ListNode l2) {
  ListNode start = new ListNode(0), iter = start;
  while (l1 != null && l2 != null) {
    if (l1.val < l2.val) {
      iter.next = l1;
      l1 = l1.next;
    } else {
      iter.next = l2;
      l2 = l2.next;
    }
    iter = iter.next;
  }

  if (l1 != null) {
    iter.next = l1;
  }

  if (l2 != null) {
    iter.next = l2;
  }

  return start.next;
}
```

逻辑如下：

- 使用递归的方式，将链表一分为二，再重新组合
- 重组的过程也是排序的过程
- 对应到前面说的三步骤为：分治（一分为二），重组（解决&合并）

## 4 例题-判断链表是否是回文

回文的意思类似：1-2-2-1；1-3-1；2-2-3-3-2-2；

判断链表是否回文最佳的比较方式是找到中点，然后从中点开始往两侧遍历，检查节点是否一致。

逻辑为：

- 找到中点
- 遍历并比较

这里的主要问题是链表是单向的，中点向前遍历并没有指针，那么这里「找到中点」的过程需要进行扩展

- 找到中点，过程中对前半部分链表在遍历同时进行反转

这样操作之后，在找到中点时，两侧的链表指针也是按照我们希望的方向进行的。

代码如下：

```java
public boolean isPalindrome(ListNode head) {
  if (head == null || head.next == null) {
    return true;
  }

  ListNode prev = null, iter = head, mid = head;
  while (iter != null && iter.next != null) {
    iter = iter.next.next;
    ListNode next = mid.next;
    mid.next = prev;
    prev = mid;
    mid = next;
  }

  ListNode secondHalf = null;
  if (iter == null) {
    secondHalf = mid;
  } else {
    secondHalf = mid.next;
  }
  mid = prev;

  while (mid != null && secondHalf != null) {
    if (mid.val != secondHalf.val) {
      return false;
    }

    mid = mid.next;
    secondHalf = secondHalf.next;
  }

  return true;
}
```

## 5 总结

Easy和Medium部分的题目基本上掌握上述概念，都能很好的完成。而Hard类型的题目，往往是借助并且超出链表本身的概念，没有太多共性的部分，需要一定的随机应变和思维能力，只能祝好运了。
