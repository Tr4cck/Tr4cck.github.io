---
title: 「Golang」Go Tour
date: 2021-06-10 17:04:27
tags:
- CS
categories:
- TECHNOLOGY
---


## Basic Syntax

关于为什么声明变量时要和C有所区别，参见这篇文章[Go's Declaration Syntax](https://blog.go-zh.org/gos-declaration-syntax)

## Basic Type

Go 的基本类型有

```go
bool

string

int  int8  int16  int32  int64
uint uint8 uint16 uint32 uint64 uintptr

byte // uint8 的别名

rune // int32 的别名
    // 表示一个 Unicode 码点

float32 float64

complex64 complex128
```

本例展示了几种类型的变量。 同导入语句一样，变量声明也可以“分组”成一个语法块。

`int`, `uint` 和 `uintptr` 在 32 位系统上通常为 32 位宽，在 64 位系统上则为 64 位宽。 当你需要一个整数值时应使用 `int` 类型，除非你有特殊的理由使用固定大小或无符号的整数类型。

## 零值

没有明确初始值的变量声明会被赋予它们的 **零值**。

零值是：

- 数值类型为 `0`，
- 布尔类型为 `false`，
- 字符串为 `""`（空字符串）。

## 数值常量

一个在全局声明的数值常量由上下文决定其类型

## 循环

如果省略循环条件，该循环就不会结束，因此无限循环可以写得很紧凑。

```go
package main

import "fmt"

func main() {
    for{
        fmt.Println("rxyyds")
    }
}
```

## 选择

### if

同 `for` 一样， `if` 语句可以在条件表达式前执行一个简单的语句。

该语句声明的变量作用域仅在 `if` 之内。

**但是**，在 `if` 的简短语句中声明的变量同样可以在任何对应的 `else` 块中使用。

### switch

Go 只运行选定的 case，而非之后所有的 case。 实际上，Go 自动提供了在这些语言中每个 case 后面所需的 `break` 语句。

**除非以 `fallthrough` 语句结束，否则分支会自动终止。**

Go 的另一点重要的不同在于 switch 的 case 无需为常量，且取值不必为整数。（这一个特性非常关键，使得switch的使用频率会大大增加）

没有条件的 switch 同 `switch true` 一样。（就类似于`for`不写条件得到一个无限循环）

## defer

### How?

defer 语句会将函数推迟到外层函数返回之后执行。

推迟调用的函数其参数会立即求值，但直到外层函数返回前该函数都不会被调用。

### defer栈

推迟的函数调用会被压入一个栈中。当外层函数返回时，被推迟的函数会按照后进先出的顺序调用。

```go
package main

import "fmt"

func main() {
	fmt.Println("counting")

	for i := 0; i < 10; i++ {
		defer fmt.Println(i)
	}

	fmt.Println("done")
}
```

输出结果：

```
counting
done
9
8
7
6
5
4
3
2
1
0
```

### defer-panic-and-recover

看看这个吧！[defer-panic-recover](https://blog.go-zh.org/defer-panic-and-recover)

```go
func CopyFile(dstName, srcName string) (written int64, err error) {
    src, err := os.Open(srcName)
    if err != nil {
        return
    }
    defer src.Close()

    dst, err := os.Create(dstName)
    if err != nil {
        return
    }
    defer dst.Close()

    return io.Copy(dst, src)
}
```

综合例子

```go
package main

import "fmt"

func main() {
    f()
    fmt.Println("Returned normally from f.")
}

func f() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered in f", r)
        }
    }()
    fmt.Println("Calling g.")
    g(0)
    fmt.Println("Returned normally from g.")
}

func g(i int) {
    if i > 3 {
        fmt.Println("Panicking!")
        panic(fmt.Sprintf("%v", i))
    }
    defer fmt.Println("Defer in g", i)
    fmt.Println("Printing in g", i)
    g(i + 1)
}
```

```
Calling g.
Printing in g 0
Printing in g 1
Printing in g 2
Printing in g 3
Panicking!
Defer in g 3
Defer in g 2
Defer in g 1
Defer in g 0
panic: 4
 
panic PC=0x2a9cd8
[stack trace omitted]
```

## 指针

Go 拥有指针。指针保存了值的内存地址。

类型 `*T` 是指向 `T` 类型值的指针。**其零值为 `nil`。**

```
var p *int
```

`&` 操作符会生成一个指向其操作数的指针。

```
i := 42
p = &i
```

`*` 操作符表示指针指向的底层值。

```
fmt.Println(*p) // 通过指针 p 读取 i
*p = 21         // 通过指针 p 设置 i
```

这也就是通常所说的“间接引用”或“重定向”。

**与 C 不同，Go 没有指针运算。**

例子：

```go
package main

import "fmt"

func main() {
	i, j := 42, 2701

	p := &i         // 指向 i
	fmt.Println(*p) // 通过指针读取 i 的值
	*p = 21         // 通过指针设置 i 的值
	fmt.Println(i)  // 查看 i 的值

	p = &j         // 指向 j
	*p = *p / 37   // 通过指针对 j 进行除法运算
	fmt.Println(j) // 查看 j 的值
}
```

结果：

```
42
21
73
```

## 结构体

用一个例子来说明

```go
package main

import "fmt"

type Vertex struct {
	X int
	Y int
}

func main() {
	fmt.Println(Vertex{1, 2}) // 大括号来初始化
}
```

结构体字段可以通过**点号**来访问，也可以通过**结构体指针**来访问。

如果我们有一个指向结构体的指针 `p`，那么可以通过 `(*p).X` 来访问其字段 `X`。不过这么写太啰嗦了，所以语言也允许我们使用**隐式间接**引用，直接写 `p.X` 就可以。

## 多线程

### 协程

Goroutine是一个协程，它与所处的函数是分开运行的，并且具有背后运行机制。即如果它所处的函数已经结束运行，不管还有多少Goroutine没有结束，都会被终止运行

```go
package main

import(
	"fmt"
	"time"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	wg.Add(2)
	
	c := make(chan string)
	
	go func() {
		count(5, "ysh")
		wg.Done()
	}()
	go func(){
		count(3, "bobo")
		wg.Done()
	}()
	wg.Wait()
}

func count(n int, baby string){
	for i := 0; i < n; i++ {
		fmt.Println(i+1, baby)
		time.Sleep(time.Second * 1)
	}
}
```

### 线程通信

#### 一般语言——内存共享来交流

通常一个具有多线程能力的编程语言都需要线程之间进行交流，而一般是通过共享内存来交流，即各个线程共同操纵同一片内存区域，并且开发者为了避免同时操纵内存造成的错误，还发明了线程锁，包括最简单的自旋锁。

#### Golang——交流来共享内存

go语言并不是这么操作的，它是通过线程间的一个channel来交流，（可以简单把它理解为自带的一个锁），这就表现为：

**发送一条消息，代码被阻塞，直到有人收听；收听一条消息，代码被阻塞，直到有人发送**

```go
package main

import (
	"fmt"
    "time"
)

func main() {
    c := make(chan string)
    go count(5, "hs", c)
    for message := range c {
        fmt.Println(message)
    }
}

func count(n int, animal string, c chan string) {
    for i := 0; i < n; i++ {
        c <- animal
        time.Sleep(time.Millisecond * 500)
    }
	close(c)
}
```

关于range可以查看这个：[range](https://github.com/golang/go/wiki/Range)

#### 如何管理多个channel？

首先说一下为什么要涉及这个问题，你可能会觉得我难道不能写一个循环，先接收一条消息，再接收一条消息吗？确实，你确实可以，但这么做的代价也很容易想到——**因为channel的阻塞特性，如果两个线程的运行时间相差非常大的话，就会造成非常严重的时间损耗**

我们可以使用select语句，它会在我们提前给定好的channel中选择没有被阻塞的来运行

```
package main

import (
	"fmt"
	"time"
)

func main() {
	c1 := make(chan string)
	c2 := make(chan string)
	go func() {
		c1 <- "hs"
		time.Sleep(time.Millisecond * 500)
	}()
	go func() {
		c2 <- "dbt"
		time.Sleep(time.Millisecond * 2000)
	}()
	for {
		select {
			case msg := <- c1:
				fmt.Println(msg)
			case msg := <- c2:
				fmt.Println(msg)
		}
	}
}
```

## map的正确使用姿势

直接查看[map](https://www.jianshu.com/p/726688f6539c)

## sort排序

参考[sort](https://www.jianshu.com/p/f79aaf39ded8)

## slice切片

参考[slice](https://www.jianshu.com/p/ca4891ad6e4f)

### 切片就像数组的引用

切片并不存储任何数据，它只是描述了底层数组中的一段。

更改切片的元素会修改其底层数组中对应的元素。

与它共享底层数组的切片都会观测到这些修改。

```go
package main

import "fmt"

func main() {
	names := [4]string{
		"John",
		"Paul",
		"George",
		"Ringo",
	}
	fmt.Println(names)

	a := names[0:2]
	b := names[1:3]
	fmt.Println(a, b)

	b[0] = "XXX"
	fmt.Println(a, b)
	fmt.Println(names)
}

// 输出结果
//[John Paul George Ringo]
//[John Paul] [Paul George]
//[John XXX] [XXX George]
//[John XXX George Ringo]
```

### 切片文法

切片文法类似于没有长度的数组文法。

这是一个数组文法：

```go
[3]bool{true, true, false}
```

下面这样则会创建一个和上面相同的数组，然后构建一个引用了它的切片：

```go
[]bool{true, true, false}
```

```go
package main

import "fmt"

func main() {
	q := []int{2, 3, 5, 7, 11, 13}
	fmt.Println(q)

	r := []bool{true, false, true, true, false, true}
	fmt.Println(r)

	s := []struct {
		i int
		b bool
	}{
		{2, true},
		{3, false},
		{5, true},
		{7, true},
		{11, false},
		{13, true},
	}
	fmt.Println(s)
}
```

### 切片的长度与容量

切片拥有 **长度** 和 **容量**。

切片的长度就是它所包含的元素个数。

切片的容量是从它的第一个元素开始数，到其底层数组元素末尾的个数。

切片 `s` 的长度和容量可通过表达式 `len(s)` 和 `cap(s)` 来获取。

你可以通过重新切片来扩展一个切片，给它提供足够的容量。试着修改示例程序中的切片操作，向外扩展它的容量，看看会发生什么。

### nil 切片

切片的零值是 `nil`。

nil 切片的长度和容量为 0 且没有底层数组。

### 用 make 创建切片

切片可以用内建函数 `make` 来创建，这也是你创建动态数组的方式。

`make` 函数会分配一个元素为零值的数组并返回一个引用了它的切片：

```go
a := make([]int, 5)  // len(a)=5
```

要指定它的容量，需向 `make` 传入第三个参数：

```go
b := make([]int, 0, 5) // len(b)=0, cap(b)=5

b = b[:cap(b)] // len(b)=5, cap(b)=5
b = b[1:]      // len(b)=4, cap(b)=4
```

### 切片的切片

切片可包含任何类型，甚至包括其它的切片。

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	// 创建一个井字板（经典游戏）
	board := [][]string{
		[]string{"_", "_", "_"},
		[]string{"_", "_", "_"},
		[]string{"_", "_", "_"},
	}

	// 两个玩家轮流打上 X 和 O
	board[0][0] = "X"
	board[2][2] = "O"
	board[1][2] = "X"
	board[1][0] = "O"
	board[0][2] = "X"

	for i := 0; i < len(board); i++ {
		fmt.Printf("%s\n", strings.Join(board[i], " "))
	}
}
```

### 向切片追加元素

为切片追加新的元素是种常用的操作，为此 Go 提供了内建的 `append` 函数。内建函数的[文档](https://go-zh.org/pkg/builtin/#append)对此函数有详细的介绍。

```go
func append(s []T, vs ...T) []T
```

`append` 的第一个参数 `s` 是一个元素类型为 `T` 的切片，其余类型为 `T` 的值将会追加到该切片的末尾。

`append` 的结果是一个包含原切片所有元素加上新添加元素的切片。

当 `s` 的底层数组太小，不足以容纳所有给定的值时，它就会分配一个更大的数组。返回的切片会指向这个新分配的数组。

```go
package main

import "fmt"

func main() {
	var s []int
	printSlice(s)

	// 添加一个空切片
	s = append(s, 0)
	printSlice(s)

	// 这个切片会按需增长
	s = append(s, 1)
	printSlice(s)

	// 可以一次性添加多个元素
	s = append(s, 2, 3, 4)
	printSlice(s)
}

func printSlice(s []int) {
	fmt.Printf("len=%d cap=%d %v\n", len(s), cap(s), s)
}
```



[Go 切片：用法和本质](https://blog.go-zh.org/go-slices-usage-and-internals)

## range用法

参考[range](https://github.com/golang/go/wiki/Range)以及[Golang中range的使用方法及注意事项](https://studygolang.com/articles/12958)

## unsafe.Pointer VS. uintptr

> 可以在go官网下载go的source，接着找到$GOROOT（现在没有这个环境变量了，它其实不应该有。可用命令go env命令得到）下src下的unsafe中的unsafe.go，然后开始你的学习

### 为什么要有这个包？

用它可以绕过Go程序内置的类型转换安全性检查，即unsafe 包会允许我们可以直接访问存储这个变量原始二进制信息的地址。在我们想绕过类型限制的时候可以使用它。

导入 unsafe 的包可能是不可移植( non-portable) 的，而且不受 Go 1 兼容性准则的保护。[Go 1 的手册清楚地说明](https://links.jianshu.com/go?to=https%3A%2F%2Fgolang.org%2Fdoc%2Fgo1compat%23expectations)，如果他们改变了实现方式，使用 unsafe 包可能会破坏你的代码。引入了 unsafe 的包可能依赖于 Go 实现的内部属性。我们保留修改实现方法的权利，这也许会破坏此类程序。我们需要记住的是，在 Go 1 中，内部实现可能会发生变化，并且我们也许会遇到类似 [issues  this ticket in Github](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fgolang%2Fgo%2Fissues%2F16769)   中所见的问题，两个 Go 版本之间的行为有略微的改变。不过， Go 的一些标准库中也在许多地方使用了 unsafe 包。

### 这个包有什么？

`type ArbitraryType int`：

它表示一个Go表达式中的任意类型，它是为了文档说明方便存在的。

`type Pointer *ArbitraryType`：

表示一个指向任意类型的指针。

`func Sizeof(x ArbitraryType) uintptr`：

接受一个任意类型的数据并返回它的字节数，类型为uintptr。**注意：**并且任何有关于x的引用都不会计算在内。

`func Offsetof(x ArbitraryType) uintptr`：

它返回一个结构体中某一字段的偏移量。

`func Alignof(x ArbitraryType) uintptr`

### 这个包怎么用？

支持四种运算：

- 任何类型的指针值都可以转换为Pointer。

- Pointer可以转换为任何类型的指针值。

- uintptr可以转换为Pointer。

- 可以将Pointer转换为uintptr。

下面给出几种使用场景

- 将 *T1 转换为指向 *T2 的指针
  假设 T2 不大于 T1 并且两者共享一个等价的内存布局，这种转换允许将一种类型的数据重新解释为其他类型的数据。

  ```go
  实现math.Float64bits:
  
  func Float64bits(f float64) uint64 {
  	return *(*uint64)(unsafe.Pointer(&f))
  }
  ```
- 将指针转换为uintptr（但不返回指针）
  将指针转换为uintptr会产生值的内存地址，并视作一个整数。这种uintptr的通常用途是打印它。
  
- 将uintptr转换回Pointer通常是无效的
  uintptr 是一个整数，而不是一个引用。
  将指针转换为 uintptr 会创建一个整数值，它没有指针语义。
  即使 uintptr 持有某个对象的地址，垃圾收集器不会更新那个 uintptr 的值，如果对象移动，那么 uintptr 也不会保留对象。
  
- 使用算术将指针转换为uintptr并返回
  如果p指向一个已分配的对象，则可以通过该对象进行索引，即通过转换为 uintptr，添加一个偏移量，然后转换回 Pointer。

  ```go
  p = unsafe.Pointer(uintptr(p) + offset)
  ```

  这种模式最常见的用途是访问结构体中的字段
  或数组的元素：

  ```go
  f := unsafe.Pointer(&s.f)
  f := unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + unsafe.Offsetof(s.f))
  ```

  ```go
  e := unsafe.Pointer(&x[i])
  e := unsafe.Pointer(uintptr(unsafe.Pointer(&x[0])) + i*unsafe.Sizeof(x[0]))
  ```

  以这种方式从指针中添加和减去偏移量都是有效的。
  使用 &^ 来舍入指针也是有效的，通常用于对齐。
  在所有情况下，结果必须继续指向原始分配的对象。
  
  与 C 不同，将指针移到刚好超出 C 的末尾是无效的
  
  无效：已分配空间外的端点。
  
  ```go
  var s thing
  end = unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + unsafe.Sizeof(s))
  ```
  
  无效：已分配空间外的端点。
  
  ```go
  b := make([]byte, n)
  end = unsafe.Pointer(uintptr(unsafe.Pointer(&b[0])) + uintptr(n))
  ```
  
  **无效：uintptr 不能存储在变量中**
  
  ```go
  u := uintptr(p)
  p = unsafe.Pointer(u + offset)
  ```
  
  注意指针必须指向一个已分配的对象，所以它可能不为nil。
**无效：nil 指针的转换**
  
  ```go
  u := unsafe.Pointer(nil)
  p := unsafe.Pointer(uintptr(u) + offset)
  ```
  
- 调用syscall.Syscall时将指针转换为uintptr。
  syscall 包中的 Syscall 函数直接传递它们的 uintptr 参数到操作系统，然后操作系统可能会根据调用的详细信息，将其中一些重新解释为指针。即系统调用实现隐式转换某些参数。
  如果必须将指针参数转换为 uintptr 才能用作参数，
  该转换必须出现在调用表达式本身中：
  
  ```go
  syscall.Syscall(SYS_READ, uintptr(fd), uintptr(unsafe.Pointer(p)), uintptr(n))
  ```
  
  编译器处理在参数列表中转换为 uintptr 的指针，对在汇编中实现的函数的调用：通过安排引用的分配的对象，在调用完成之前被保留并且不会移动。
  
- reflect.Value.Pointer或reflect.Value.UnsafeAddr的结果的转换
  
  包反射的名为 Pointer 和 UnsafeAddr 的 Value 方法返回类型 uintptr而不是 unsafe.Pointer 以防止调用者将结果更改为任意值而没有引用unsafe包
  
  然而，必须在调用后立即转换为指针
  
  ```go
  p := (*int)(unsafe.Pointer(reflect.ValueOf(new(int)).Pointer()))
  ```
  
- 将reflect.SliceHeader 或reflect.StringHeader Data 字段转换为Pointer 或从Pointer 转换。
  
  和前面的例子一样，反射数据结构 SliceHeader 和 StringHeader将字段 Data 声明为 uintptr 以防止调用者将结果更改为任意类型而不先导入unsafe的。然而，这意味着SliceHeader 和 StringHeader 仅在解释内容时有效。
  
  ```django
  var s string
  hdr := (*reflect.StringHeader)(unsafe.Pointer(&s)) // case 1
  hdr.Data = uintptr(unsafe.Pointer(p)) // case 6（本例）
  hdr.Len = n
  ```
  
  在这种用法中，hdr.Data 实际上是引用底层字符串头中的指针的另一种方式，而不是 uintptr 变量本身。
  一般来说，仅当 *reflect.SliceHeader 和 *reflect.StringHeader 指向实际切片或字符串（绝不是普通结构）时，我们去使用reflect.SliceHeader和reflect.StringHeader。
  程序不应声明或分配这些结构类型的变量。
  
  **无效：直接声明的标头不会将数据作为引用保存**
  
  ```go
  var hdr reflect.StringHeader
  hdr.Data = uintptr(unsafe.Pointer(p))
  hdr.Len = n
  s := *(*string)(unsafe.Pointer(&hdr)) // p 可能已经丢失
  ```
  
