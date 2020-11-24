---
title: Spring_MongDB
date: '2020/11/23 20:10:39'
updated: '2020/11/23 20:11:04'
tags: []
category:
  - Java
  - Spring
mathjax: true
---
# 1.基本类型
## a. set
```java
Query query = new Query(Criteria.where("_id").is("123"));
Update update = new Update().set("name", "munan");
mongoTemplate.updateFirst(query, update, COLLECTION_NAME);
```
<!--more-->
`set`：修改指定属性为指定数据；
`updateFirst`：修改根据query查找到的第一个实体对象。

## b. 数字相关

```java
// 数字增加或者减少
Update newCount = new Update().inc("count", 1L);

// 比较修改
Update max = new Update().max("num", 10);	// 更新为num和10中较大的数

// 乘除
Update multi = new Update().multiply("num", 2);
```

## c. 日期

```java
Update date = new Update().currentData("date");
```

# 2. field

## a. rename

```java
Update update = new Update().rename("name", "new_name");
```

如果`name`属性不存在，则不会发生更新。

## b. 增加

```java
Update update = new Update().set("new_field", "new_field");
```

新增一个字段，使用set可以完成。

当对一个不存在的文档进行更新field时候，可以用`setOnInsert`，若文档已经存在，则不会更新。

# 3. Array/List/Set

在mongodb中都转换为数组。

## a. 添加

```java
Update update = new Update().addToset("new_element", "123");

update = new Update().push("new_element", "123");
```

`addToSet`：不可以对已经存在的字段进行插入，即将数组当作Set类型处理；在后面加上each()，能够批量添加；

`push`：允许重复。

## b. 删除

```java
Update update = new Update().pull("del", "123");
```

## c. 修改

```java
Update update = new Update().set("array.1", 10L);
```

对数组array的index为1的元素修改为`long` 10。

# Note:

## a. 使用同一个Update

需要确保该Update内没有冲突的更新实体对象，即同一个update不能对实体的同一个属性进行多次修改。

## b.  使用同一个Criteria

对同一个Criteria附加chain的操作时，要注意and()等会改变原先Criteria，即在操作中是深拷贝参数的。在拼接Criteria时尽量使用new Criteria().xxx()。

