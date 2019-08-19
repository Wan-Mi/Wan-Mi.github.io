---
title: '[Godep]如何合理更新Go项目依赖'
date: 2018-01-20 01:17:33
tags: Godep
---
利用godep更新依赖库，最常见的做法是将godep.json中的依赖库版本进行更新，再`godep save`即可。但这很可能会使得母库和子库的共同依赖不同步进而可能产生报错，这才是本文想解决的问题：
Go的包管理工具godep并不像诸如iOS包管理工具CocoaPods那样可以自动统一母库A和子库B的共同依赖(如库C)。当需要对子库B进行更新时，如果单纯的在库A中更新库B，则有可能发生同一个项目A中存在不同版本的依赖库C的情况。
为了解决上述问题可进行如下操作:
1、在B路径中更新子库B（此时C可能也会跟着更新）

```
git pull
```
2、 在A路径中移除项目A中的Godeps和Vender文件夹：

```
rm -rf Godep vender
```
3、在项目A中重新生成Godeps和Vender文件夹：

```
godep save ./...
```
2和3两步看似相互抵消，实际上统一了A和B的所依赖的C的版本号，该版本为第1步中B更新后的C的版本。产生次效果的原因主要是：在`godep save`操作中godep工具首先会查找Godep目录中的Godeps.json文件，根据文件中的指定依赖生成vender；**当godep目录不存在时，会根据项目中的依赖路径寻找GOPATH中的依赖库，并据此生成Godep和Vender目录。** 本技巧正是根据上述特性实现共同依赖库版本号的统一。
