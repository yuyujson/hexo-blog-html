---
title: shell脚本
permalink: shell-jiao-ben
date: 2018-10-05 12:00:30
tags: shell
categories: linux
---

## 变量
**变量使用**

```
your_name="yuyu"
echo $your_name
echo ${your_name}
```
<!--more-->
> 等号左右不可有空格

**for循环**

```
for skill in Ada Coffe Action Java; do
    echo "I am good at ${skill}Script"
done

-- 结果
I am good at AdaScript
I am good at CoffeScript
I am good at ActionScript
I am good at JavaScript

```
> in后方是一个数组,每个元素间使用空格分开,打印完毕后需done

**只读变量**

使用 readonly 命令可以将变量定义为只读变量，只读变量的值不能被改变。

下面的例子尝试更改只读变量，结果报错：

```
myUrl="http://www.google.com"
readonly myUrl
myUrl="http://www.runoob.com"

-- 结果
/bin/sh: NAME: This variable is read only.
```

**删除变量**

```
unset variable_name
```
---
## 变量类型
1) 局部变量 局部变量在脚本或命令中定义，仅在当前shell实例中有效，其他shell启动的程序不能访问局部变量。
2) 环境变量 所有的程序，包括shell启动的程序，都能访问环境变量，有些程序需要环境变量来保证其正常运行。必要的时候shell脚本也可以定义环境变量。
3) shell变量 shell变量是由shell程序设置的特殊变量。shell变量中有一部分是环境变量，有一部分是局部变量，这些变量保证了shell的正常运行
---

## shell字符串

字符串是shell编程中最常用最有用的数据类型（除了数字和字符串，也没啥其它类型好用了），字符串可以用单引号，也可以用双引号，也可以不用引号。单双引号的区别跟PHP类似。

**单引号**
```
str='this is a string'
```
- 单引号里的任何字符都会原样输出，单引号字符串中的变量是无效的；
- 单引号字串中不能出现单独一个的单引号（对单引号使用转义符后也不行），但可成对出现，作为字符串拼接使用。
- 
**双引号**

```
your_name='runoob'
str="Hello, I know you are \"$your_name\"! \n"
echo $str

-- 输出
Hello, I know you are "runoob"! 
```

- 双引号里可以有变量
- 双引号里可以出现转义字符

**拼接字符串**

```
your_name="runoob"
# 使用双引号拼接
greeting="hello, "$your_name" !"
greeting_1="hello, ${your_name} !"
echo $greeting  $greeting_1


# 使用单引号拼接
greeting_2='hello, '$your_name' !'
greeting_3='hello, ${your_name} !'
echo $greeting_2  $greeting_3

-- 结果
hello, runoob ! hello, runoob !
hello, runoob ! hello, ${your_name} !
```

**获取字符串长度**

```
string="abcd"
echo ${#string} 
# 输出 4
```
**提取子字符串**

```
string="runoob is a great site"
echo ${string:1:4}
# 输出 unoo
```