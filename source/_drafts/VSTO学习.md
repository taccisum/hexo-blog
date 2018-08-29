---
title: VSTO学习
date: 2017-10-19 21:15:58
categories:
    - VSTO
    - VB
    - Office
    - Excel
tags:
---

## 前言


## 什么是VSTO

`VSTO`是一套用于创建自定义Office应用程序（`程序级别`或`文档级别`）的VS工具包。

使用VSTO可以创建文档级别的`定制程序`（程序代码关联到特定的文档）。不过这些代码并不像VBA定制程序一样存放在文档或模板里，而是存放在与文档相关联的`代码库（程序集）`里。当创建好新的项目，就可以通过`PIA`（Primary Interop Assemble）访问Office的对象模型。`Office PIA`允许VSTO定制程序与Office的对象模型进行交互。

此外，VSTO还提供`增强的Office对象`。这些对象在本地的Excel对象模型里是无法访问到的。

### 程序级别的插件
程序级别的插件会在应用程序启动时加载，并在应用程序关闭时卸载。
可以通过插件来定制应用程序的用户界面（例如为Excel添加菜单或菜单项）。

### 文档级别的插件

## 


VSTO支持VB2005（不是VBA）和C#。

