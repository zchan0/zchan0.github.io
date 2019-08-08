---
layout: post
title: swift compile optimization
date: 2019-07-02 15:18 +0800
draft: true
---

# 显示编译时间

# 常见的 case

## Type inference

## Ternary and Nil coalescing operator

## String/Array concate

## Pre-Calculation

# 其它的 case

## Complex Methods

*来自：[Optimizing Build Times in Swift 4](https://medium.com/rocket-fuel/optimizing-build-times-in-swift-4-dc493b1cc5f5)*

特征：一个方法做了太多的事情，不符合函数的「单一职责」原则。

修改：整理代码，分成多个函数去调用，使之更符合「单一职责」原则。

## Round()

## Closures and lazy properties

使用 `lazy` 定义变量，或者使用 closure 初始化变量



## Extraneous CGFloat conversion

