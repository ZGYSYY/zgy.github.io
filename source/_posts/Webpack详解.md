---
title: Webpack详解
date: 2020-01-03 11:02:43
tags:
- Webpack
- 前端
categories:
- 前端
- webpack
---

# nrm工具使用

简述：一个管理镜像源地址的工具。

安装：`npm i nrm -g`。

查看当前使用的镜像源地址：`nrm ls`。

切换镜像源地址：`nrm use npm`或`nrm use taobao`。

# 什么是webpack

简述：Webpack是一个前端的项目构建工具，是基于node.js开发的一个前段工具。

<!-- more -->

# Webpack安装的两种方式

1. 运行`npm i webpack -g`全局安装，这样就能在全局使用webpack命令了。
2. 在项目根目录中运行`npm i webpack --save-dev`安装到项目依赖中。