## 应用场景

1. 区块链技术
2. 服务器开发
3. 分布式、云计算

## Google为什么要创建Go语言？

1. 计算机硬件更新快，性能高，目前主流编程语言不能利用多核多CPU的优势
2. 目前缺少一个足够简洁高效的编程语言
3. c/c++运行速度很快，但是编译速度慢，而且存在内存泄露

## 类型

### 值类型

基本数据类型int系列、float系列、bool、string、数组和结构体struct

### 引用类型

指针、slice切片、map、管道chan、interface

## 变量

var定义变量，**类型放在变量名之后**

可以一次定义多个变量，包括用不同初始值定义不同类型

变量名、函数名、常量名**首字母大写，则可以被其他包来访问**，若首字母小写，则只能在本包中使用

### 简短模式

使用":="来定义

- 定义变量，同时显示初始化
- 不能提供数据类型
- 只能用在函数内部

### 退化赋值

也就是指定义变量时退化为赋值操作

前提条件：最少有一个新变量被定义，且必须是同一作用域

### 多变量赋值

先计算所有的右值，然后再依次完成赋值操作

### iota

Integer to ASCII

## 数组

==数组默认为值传递，在函数传递时会发生拷贝==

再使用数组时，一般使用指针传递数组的地址

## slice切片

引用类型，长度可以变化的数组，底层为一个struct结构体

```go
type slice struct {
   array unsafe.Pointer
   len   int
   cap   int
}
```

定义语法：var a[]int  ([]内不加长度)

### 三种定义方式

通过已有数组，将切片指向数组

```
var arr = []int{1, 2, 3, 4, 5}
arrSilce := arr[1:3]
```

通过make定义，指定len和cap，底层的数组不可见

```
var slice = make([]int, len, cap)
```
创建时同时初始化

```
var slice []int = []int{1, 2, 3, 4, 5}
```

### append

当append的操作数组超过了cap时，会新申请一个数组，再将原数组复制过去

## string

string底层是一个byte数组，可以使用切片处理

不可变

## map

key-value无序

### 三种定义方式

常规定义

```go
var dict = make(map[int]string, 10)
```

使用简短模式

```go
dict := make(map[int]string, 10)
```

定义同时初始化

```go
dict := map[int]string{
   1 : "a",
   2 : "b",
}
```

### 删除

使用delete函数删除

```
delete(dict, 1)
```

