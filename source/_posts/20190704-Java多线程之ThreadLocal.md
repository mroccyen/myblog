---
title: Java多线程之ThreadLocal
date: 2019-07-04 15:26:56
tags:
- 多线程
categories:
- Java
- 多线程
---

## 概述

ThreadLocal提供了线程的局部变量，且不会和其他线程的局部变量冲突，实现了线程的数据隔离。

## 主要接口

- set(value)
  - 获取当前线程的threadLocals变量（ThreadLocal类型）
  - 如果为null，进行初始化map
  - 否则将值存入map中，调用map的set方法，ThreadLocal作为key
- get()
  - ThreadLocal作为key
- remove()
- initialValue()

## ThreadLocalMap

ThreadLocalMap是ThreadLocal的内部类。

每个Thread维护了一个ThreadLocalMap的引用（threadLocals变量）。

### Entry

- 继承WeakReference<ThreadLocal>
- ThreadLocal的内部类
- 真正存储值的地方

###重要成员

- Entry[] table：默认大小为16
- set(ThreadLocal<?> key, Object value)
- getEntry(ThreadLocal<?> key)

### hash冲突解决方式

#### 开放式定址发法

- 线性探测法

  冲突发生时，顺序查看表中的下一单元，直到找出一个空单元或者遍历全表。如ThreadLocal采用这种方式。

- 二次探测
  冲突发生时，在表的左右进行跳跃式的探测，比较灵活。

- 伪随机探测

  具体实现时，应建立一个伪随机数发生器，（如i=(i+p) % m），并给定一个随机数做起点。

#### 链地址法（链表）

- 如HashMap的实现

#### 再哈希法

- 有多个不同的hash函数，当发生冲突时，使用第二个，第三个，…等哈希函数，计算地址，直到无冲突
- 虽然不容易发生聚集，但是增加了计算时间

#### 建立公共溢出区

将哈希表分为基本表和溢出表两部分，凡是和基本表发生冲突的元素，一律填入溢出表

### rehash

长度大于等于threshold（length的3分之2）

### resize

- 长度大于等于（threshold - threshold/4）
- 扩容为原来的2倍



## 内存泄漏

ThreadLocal在没有外部对象强引用时，GC时会将key回收，而value不会回收，这时候ThreadLocalMap中就会存在key为null但是value不为null的entry，这时线程一直运行的话，value得不到回收，发生内存泄漏。

### 解决方式

调用ThreadLocal的remove方法来显示清理key为null的元素。

### 根本原因

ThreadLocalMap的生命周期和线程是一样的，如果没有手动删除key，对应的value就会内存泄漏。



## 应用场景

- 单个线程
- 线程上下文信息存储
- 数据库连接
- session管理