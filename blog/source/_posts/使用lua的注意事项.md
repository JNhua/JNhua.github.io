---
title: 使用lua的注意事项
date: '2021/9/20 17:45:16'
updated: '2021/9/20 18:06:06'
tags: []
category:
  - 编程
  - lua
mathjax: true
toc: false
abbrlink: 7fd42f65
---
1. #table 可以用来判断一个数组的长度，但需要注意的是，若table中包含nil，则不可使用。
<!--more-->

2. table.sort同理，需要排序的table必须是1到n连续的，中间不能有nil。

3. 尽量使用局部变量，函数function也是如此，因为在lua里函数也是一个变量。局部变量的存取会更快，且生命周期外就会被释放。

4. 避免使用table.insert()
方法1：
```lua
local a = {}
local table_insert = table.insert
for i = 1,100 do
   table_insert( a, i )
end
```
方法2：
```lua
local a = {}
for i = 1,100 do
   a[#a+1] = i
end
```
方法3：
```lua
local a = {}
for i = 1,100 do
   a[i] = i
end
```
推荐使用方法3。其中方法3远优于1,2。而方法1略优于方法2。

5. ipairs和pairs的区别
    1. ipairs遇到nil会停止，pairs会输出nil值然后继续下去
    2. ipairs顺序输出table中的值。而pairs会乱序(键的哈希值)输出键值对。