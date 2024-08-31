---
layout:       post
title:        "Go切片（slice）的误解和坑"
author:       "KC"
header-style: text
catalog:      true
tags:
    - Golang
---

切片（slice）是Go语言的内置数据类型，本质上实现的是动态数组的功能，是对数组的抽象。

我们知道数组是相同元素的有序集合，在内存中占据了连续的空间，是编程语言中使用频繁的数据类型。实现复杂数据结构的基础类型。数据可以通过索引快速访问，同时由于其占据连续内存空间，遍历和批量读取的速度比较快。数组元素按照顺序连续存储，每个元素占据固定大小的位置，不需要额外保存其他位置信息，节省内存资源。

由于数组这些特征，初始化以后，操作系统就会分配一块大小固定的内存给数组，数组大小是无法更改的。任何增加和删除元素的操作都会导致数组重新进行内存分配。

在实际使用数组过程中，很多情况我们是需要对数组大小进行调整，或者在初始化数组时无法确定需要使用的大小，如果直接使用数组，当增加或者删除元素时，就需要写大量冗余的代码。动态数组就是屏蔽了这些细节，数组增加元素时会自动根据需要重新分配内存，不需要程序员去关心这些内容。

slice本质上是动态数组，但是它也有自己的特征:

- slice会自动扩容，但是不允许删除元素

- 多个slice可以共享相同的底层数组，可以重叠、部分共享

## 知识点：slice扩容算法

如果只是声明slice，则底层数组不会被创建。

var values []int

如果使用make函数或者赋值的方式，底层数组才会被创建

values :=[]int{}

values := make([]int, 0)

无论1和2，哪种方式，都可以直接使用append追加元素，不会报错。当底层数组没有创建时，会自动创建，不会报错。

使用append追加元素时，追加后的总元素数量大于容量时触发自动扩容。

当容量小于256时，直接将旧的容量乘以2作为新的容量大小。

当容量大于256时，新的容量 =1.25*旧容量 + 192

自动扩容时，有一个阀值256是扩容速度的阀值，在老版本的Go中这个数字是1024，后面改成了256，从乘以2到乘以1.25转变的过程中，还有一个192的数字，这个数字是一个缓慢的过度，扩容速度并不是以下从乘以2降到乘以1.25，当容量达到一定程度时，192这个额外增加的容量就没有那么重要了，扩容速度就近似等于1.25.

这个数字是怎么来的不清楚，应该是Go团队基于一些数据推导出来的优化方案？具体的代码，以1.23为例（已经删除注释）：
```
func nextslicecap(newLen, oldCap int) int {
    newcap := oldCap
    doublecap := newcap + newcap
    if newLen > doublecap {
        return newLen
    }
    const threshold = 256
    if oldCap < threshold {
        return doublecap
    }
    for {
        newcap += (newcap + 3*threshold) >> 2
        if uint(newcap) >= uint(newLen) {
            break
        }
    }

    if newcap <= 0 {
        return newLen
    }
    return newcap
}
```

## 知识点：slice是不是引用类型

slice究竟是不是引用类型？，我们直接看一下slice的结构体：

```
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

slice中有三个字段：

- array是一个指针，这是引用类型，它指向的是一个数组的起始位置

- len表示切片当前元素的数量

- cap表示切片的容量

其实直接从这个定义就能判断，slice不是引用类型。

首先，slice是内置的结构体，&slice才是引用。那么为什么那么多人说slice是引用类型，我认为主要是两个方面。

Go将slice的零值定义为nil

使用占位符打印slice地址的时候，打印的是slice指向的数组的位置。

直接看下面两个知识点，我对这两个问题的解释

### 知识点：slice的零值问题

```
func main() {
	var v1 []int
  v2 := []int{}
  v3 := make([]int, 0)
  fmt.Println(v1==nil,v2==nil,v3==nil)
}

// true false false
```

只有 v1 是 nil，v2和v3不是nil，而是值为0的空数组。因为在Go的语法中对切片的判断会直接转化成对切片中指向数组的那个指针的判断，给人的印象就是slice就是引用类型。


## 知识点：slice的位置信息

这个和上面那个知识点本质都是因为Go将slice的位置转化为slice指向的底层数组的位置！并不是slice的位置！

```
func main() {
  values := []int{1, 2, 3, 4}
	values2 := values
	values3 := values[1:2]
	fmt.Logf("%p", values)
	fmt.Logf("%p", values2)
	fmt.Logf("%p", values3)
}
```

## 坑1：对clice类型的误解导致的坑——slice作为参数传入到函数中

如果认为slice就是引用类型，那么可能会犯这样的错误，我们想要将切片追加三个元素，将切片作为参数传入到初始化的函数中。但是结果可能不是我们需要的：

```
func Add3Item(data []int) {
   for i := range 3 {
      data = append(data, i)
   }
}

func main() {
  values := []int{100,200}
  Add3Item(values)
  fmt.Println(values)
  // 这段代码的输出是 [100,200]，不是期待的[100,200,0,1,2]。
}
```

这段代码的输出是 [100,200]，不是期待的[100,200,0,1,2]。

修复的方式是返回一个新的切片：

```
func Add3Item(data []int) []int {
   for i := range 3 {
      data = append(data, i)
   }
  return data
}

func main() {
  values := []int{100,200}
  values = Add3Item(values)
  fmt.Println(values)
  // [100,200,0,1,2]
}
```

或者使用&引用数组：

```
func Add3Item(data *[]int) {
   for i := range 3 {
      *data = append(*data, i)
   }
  return
}

func main() {
  values := []int{100,200}
  values = Add3Item(&values)
  fmt.Println(values)
  // [100,200,0,1,2]
}
```

## 坑2：多个切片指向相同的底层数组导致的坑

虽然slice不是引用类型，但是由于不同的slice可以指向相同的底层数组，所以在slice赋值或者部分赋值的情况下，多个slice指向相同的底层数组，一个slice更改元素会导致其他的slice的值也发生变化。

```
func change(data []int) {
	data[0],data[1] = 0,0
}

func main() {
  values := []int{100,200}
  change(values)
  fmt.Println(values)
  // [0,0]
}
```

## 坑3：copy内置函数拷贝slice，拷贝的长度取决于目标slice的长度

这个函数的实现是比较坑的，如果目标slice的长度为0，则copy函数复制0个值。代码如下：

```
func main() {
  source := []int{100,200}
  dest := []int{}
  copy(dest, source)
  fmt.Println(dest)
  // []
}
```

这段代码dest输出为空 []。如果要全量拷贝，需要进行如下修改：

```
func main() {
  source := []int{100,200}
  dest := make([]int, len(source))
  copy(dest, source)
  fmt.Println(dest)
  // [100,200]
}
```

或者我们直接使用slices标准库中的拷贝函数,slices.Clone。

## 坑4：slice零值为空的误解

由于slice的零值是其指向底层数组的指针决定，所以slice的零值是nil，但是实际上有一些slice指向的底层数组已经初始化，那么值就是[]，不是nil。在使用中需要注意，不能通过 if slice == nil 来判断slice是否值为空，而应该使用len(slice)函数判断。


```
func main() {
    var values []int
    values2 := make([]int,0)
  	fmt.Println(values==nil, values2 == nil)
    // true, false
   	fmt.Println(len(values)==0, len(values2) == 0
    // true, true
}
```

