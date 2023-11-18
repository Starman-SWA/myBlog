---
title: A Tour of Go学习笔记
date: 2021-07-29 15:29:10
tags:
- Go
categories:
- Go
---

# 下载A Tour of Go本地中文版

A Tour of Go的中文网页 https://tour.go-zh.org/ 提示隐私错误，无法访问，可安装本地版进行离线学习。

1. 安装Go

[Downloads - The Go Programming Language (golang.org)](https://golang.org/dl/)

2. 配置环境变量

Go有两个环境变量：`$GOROOT`和`$GOPATH`，前者是Go的安装目录，后者是Go的工作区目录。

3. 设置代理

```powershell
go env -w GOPROXY=https://goproxy.cn,direct
set GO111MODULE=on
```

4. 下载A Tour of Go中文版

```powershell
go get -u github.com/Go-zh/tour
```

5. 运行

```powershell
cd D:\Documents\source\go\pkg\mod\github.com\!go-zh\tour@v0.0.0-20210601082505-f4baf0dba327
go run .
```



# Basics

## 包、变量和函数

### 包

Go的主程序从package main中的func main开始执行。

### 导入

```go
package main

import (
	"fmt"
	"math"
)

func main() {
	fmt.Printf("Now you have %g problems.\n", math.Sqrt(7))
}
```

使用import语句导入包，包名要加引号，括号实现多个包的导入，也可通过多条单独的import语句实现。

导入包中的子函数时用”/“作为分隔符：

```go
...
import "math/rand"
...
fmt.Println("My favorite number is", rand.Intn(10))
```

### 导出名

导出名首字母必须大写，例如`math.Pi`。

### 函数

函数定义格式如下，注意类型在变量名之后：

```go
func add(x int, y int) int {

	return x + y

}
```

`x int, y int`可简写成`x, y int`。

### 多值返回

使用括号括起来：

```go
func swap(x, y string) (string, string) {
	return y, x
}
```

### 命名返回值

Go的返回值能够被命名，可对返回值变量直接赋值，然后省略return语句后面的变量名。

```go
func split(sum int) (x, y int) {
	x = sum * 4 / 9
	y = sum - x
	return
}
```

注意，**return语句本身不能省略**，只有无返回值的函数能够不写return语句。

### 变量

`var c, python, java bool`

注意没有`var i int, j bool`这种写法

变量可在函数内声明，也可在函数外（包级别）声明。

函数内外声明的变量均进行了默认初始化。

### 变量的初始化

```go
var i, j int = 1, 2 // 初始化必须在等号后面一次性写出
var c, python, java = true, false, "no!" // 变量类型明确时，可不显式写出
```

注意不能`var i = 1, j = 2`。

### 短变量声明

可用`:=`代替var声明：`k := 3`，但是`:=`不能在函数外使用。

### 基本类型

- bool

- string

- int  int8  int16  int32  int64
- uint uint8 uint16 uint32 uint64 uintptr
- byte // uint8 的别名

- rune // int32 的别名
    // 表示一个 Unicode 码点

- float32 float64

- complex64 complex128

`int`, `uint` 和 `uintptr` 在 32 位系统上通常为 32 位宽，在 64 位系统上则为 64 位宽。

### 零值

未被初始化的变量会被赋值为0、false或""。

### 类型转换

```go
var i int = 42
var f float64 = float64(i)
// 或者f := float64(i)
```

Go要求显式类型转换。

### 类型推导

当使用`:=`声明变量时，Go会根据右侧的变量类型进行类型推导。

### 常量

```go
const Pi = 3.14
```

常量不能使用`:=`声明。

### 数值常量

```go
const (
	// 将 1 左移 100 位来创建一个非常大的数字
	// 即这个数的二进制是 1 后面跟着 100 个 0
	Big = 1 << 100
	// 再往右移 99 位，即 Small = 1 << 1，或者说 Small = 2
	Small = Big >> 99
)
```

数值常量是高精度的值，一个未指定类型的常量由上下文来决定其类型。
注意到可用括号同时声明多个常量。

## 流程控制语句：for、if、else、switch和defer

### for

```go
for i := 0; i < 10; i++ {
    sum += i
}
```

for是Go中唯一的循环结构。for语句无需使用小括号，而花括号是必须的。for循环的初始化语句和后置语句是可选的。

### for是Go中的”while“

```go
for sum < 100 {
	sum += sum
}
```

for可以只写循环条件部分，省略分号，此时相当于while。

### if 

与for类似，无需小括号，但必须使用花括号。

### if的简短语句

```go
if v := math.Pow(x, n); v < lim {
	return v
}
```

if可以在判断条件之前执行一个简单的语句，适用于对某个变量进行赋值之后需要立即进行判断的情况。

**注意：同for循环类似，此处声明的变量只在if中可见，if之外不可见。**

### if和else

同样可以使用简短的声明语句，且声明的变量在所有else分支中均可见。else if当中也可以继续声明变量。

### 练习：循环和函数

牛顿法实现平方根函数：
```go
package main

import (
	"fmt"
)

func Sqrt(x float64) float64 {
	z := 1.0
	times := 10
	fmt.Println("x ==", x)
	for count := 0; count < times; count++ {
		z -= (z * z - x) / (2 * z)
		fmt.Println("Times:", count, "result:", z)
	}
	return z
}

func main() {
	fmt.Println(Sqrt(2))
}
```

### switch

```go
switch os := runtime.GOOS; os {
case "darwin":
	fmt.Println("OS X.")
case "linux":
	fmt.Println("Linux.")
default:
	// freebsd, openbsd,
	// plan9, windows...
	fmt.Printf("%s.\n", os)
}
```

Go中的switch特征：
- case后面默认带了break，除非以fallthrough语句结束
- case值无需为常量，不必为整数

### switch的求值顺序

switch从上到下逐个匹配case语句，匹配成功时停止。对于如下语句，在`i==0`时函数f()不会被调用：

```go
switch i {
case 0:
case f():
}
```

### 没有条件的switch

相当于`switch true`，即case当中的求值结果为`true`时执行。可以用于代替复杂的if-else。

### defer

defer语句会将函数推迟到外层函数返回后执行。注意，**参数会被立即求值**，但函数会被推迟调用。

```go
func main() {
	defer fmt.Println("world")

	fmt.Println("hello")
}
```

### defer栈

被推迟调用的函数会被压入一个栈中，外层函数返回时，依次弹出并访问栈顶元素。

```go
func main() {
	fmt.Println("counting")

	for i := 0; i < 10; i++ {
		defer fmt.Println(i)
	}

	fmt.Println("done")
}
```

## 更多类型：struct、slice和映射

### 指针

声明指针：`var p *int`

使用指针：
```go
i := 42
p = &i
fmt.Println(*p)
*p = 21
```

但是，与C语言不同，Go没有指针运算，不能对指针进行`p++`之类的操作。

### 结构体

```go
type Vertex struct {
	X int
	Y int
} // 声明方式为type name struct {}

func main() {
	fmt.Println(Vertex{1, 2}) // 匿名struct
}
```

### 结构体字段

```go
v := Vertex{1, 2} // 使用匿名struct进行初始化，自动进行类型推断
v.X = 4 // 使用点访问成员
fmt.Println(v.X)
```

### 结构体指针

```go
func main() {
	v := Vertex{1, 2}
	p := &v // 简单声明
	var q *Vertex // 显式声明
	q = &v
	p.X = 1e9
	fmt.Println(v)
	fmt.Println(*q)
}
```

```
{1000000000 2}
{1000000000 2}
```

### 结构体文法

结构体文法通过直接列出字段的值来分配一个结构体（即匿名结构体），可以通过“字段名+冒号”来列出部分字段，且无需按顺序列出，例如可执行语句`v2 := Vertex{X: 1}`，此时Y: 0被隐式地赋予。

### 数组

类型说明`T[n]`表示含有n个T类型元素的数组，如`var a [10]int`。数组的长度是类型的一部分，不能修改。数组声明时会分配空间，所有元素会被默认初始化。

```go
var a [2]int
fmt.Println(a[0], a[1])
```

```
0 0
```

### 切片

切片也是一个变量，类型说明`[]T`表示T类型的切片。切片的范围由一个左闭右开区间指定：`var s []int = primes[1:4]`。

### 切片就像数组的引用

**切片并不存储数据，更改切片元素会修改底层数组中对应的元素。**

### 切片文法

切片文法类似于一个没有长度的数组文法。注意切片的类型说明符中不能带长度。

`[]bool{true, true, false}`语句创建了一个数组，然后构建了一个引用它的切片。

### 切片的默认行为

进行切片时，可省略上下界。以下切片是等价的：
```go
a[0:10]
a[:10]
a[0:]
a[:]
```

### 切片的长度和容量

切片的长度就是它所包含元素的个数；切片的容量是从它的第一个元素开始数，到其底层数组末尾的个数。

切片`s`的长度和容量可通过表达式`len(s)`和`cap(s)`来获取。

只要具有足够的容量，可以通过重新切片来扩展一个切片。

```go
package main

import "fmt"

func main() {
	s := []int{2, 3, 5, 7, 11, 13}
	printSlice(s)

	// 截取切片使其长度为 0
	s = s[:0]
	printSlice(s)

	// 拓展其长度
	s = s[:4]
	printSlice(s)

	// 舍弃前两个值
	s = s[2:]
	printSlice(s)
}

func printSlice(s []int) {
	fmt.Printf("len=%d cap=%d %v\n", len(s), cap(s), s)
}
```

```
len=6 cap=6 [2 3 5 7 11 13]
len=0 cap=6 []
len=4 cap=6 [2 3 5 7]
len=2 cap=4 [5 7]
```

可见截取一个切片的前部时，切片的容量不变。但截取切片后部，即舍弃切片前部时，切片的容量会减小。

### nil切片

切片的零值是`nil`。

nil 切片的长度和容量为 0 且没有底层数组。

声明nil切片：`var s []int`。

### 用make创建切片

`make`函数会分配一个元素为零值的数组并返回一个引用了它的切片：

```go
a := make([]int, 5)  // len(a)=cap(a)=5
```

要指定它的容量，需向make传入第三个参数：

```go
b := make([]int, 0, 5) // len(b)=0, cap(b)=5

b = b[:cap(b)] // len(b)=5, cap(b)=5
b = b[1:]      // len(b)=4, cap(b)=4
```

注意，二次切片的len有可能比父切片大，例如：

```go
func main() {
	a := make([]int, 5)
	printSlice("a", a)

	b := make([]int, 0, 5)
	printSlice("b", b)

	c := b[:2]
	printSlice("c", c)

	d := c[2:5]
	printSlice("d", d)
}
```

```
a len=5 cap=5 [0 0 0 0 0]
b len=0 cap=5 []
c len=2 cap=5 [0 0]
d len=3 cap=3 [0 0 0]
```

c的长度只有2，但是d取了一个长度为3的切片，此时可以越过c的长度，继续从底层数组中取第三个元素。

### 切片的切片

```go
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
```

### 向切片追加元素

`func append(s []T, vs ...T) []T`

s为待追加的切片，vs为要追加的值，返回新切片。当 s 的底层数组太小，不足以容纳所有给定的值时，它就会分配一个更大的数组。返回的切片会指向这个新分配的数组。

注意，关于追加值是否会改变原底层数组的值，经过我的测试，分为两种情况：
- 如果底层数组容量足够，则追加值直接覆盖底层数组原来的值
- 如果底层数组容量不足，则不会改变原数组，而会分配一个新的数组，切片指向新数组

### Range

for 循环的 range 形式可遍历切片或映射。

当使用 for 循环遍历切片时，每次迭代都会返回两个值。第一个值为当前元素的下标，第二个值为该下标所对应元素的一份**副本**。

```go
var pow = []int{1, 2, 4, 8, 16, 32, 64, 128}

func main() {
	for i, v := range pow {
		fmt.Printf("2**%d = %d\n", i, v)
	}
}
```

Go中不允许存在声明但未使用的变量，若不需要使用下标或者元素值，可以用下划线代替变量名忽略它；如果只需要使用下标，可以直接不写第二个变量和逗号。

```go
for i, _ := range pow
for _, value := range pow
for i := range pow
```

当使用range遍历映射时，第一个值为key，第二个值为value。

### 练习：切片

创建一个二维切片，作为存储图像灰度值的矩阵，然后依次填入每个灰度值。

```go
func Pic(dx, dy int) [][]uint8 {
	slice := make([][]uint8, dy)
	for i := 0; i < dy; i++ {
		slice[i] = make([]uint8, dx) // (*)
	}
	
	for i := 0; i < dy; i++ {
		for j := 0; j < dx; j++ {
			slice[i][j] = uint8(i) ^ uint8(j)
		}
	}
	
	return slice
}
```

我一开始把(\*)行的`=`写成了`:=`，出现报错，原因是slice[i]是一个已存在的元素，为nil切片，而`:=`只能用于声明新的变量。

### 映射（map）
映射将key映射到value。零值为nil，既没有key，也不能添加key。make函数会返回给定类型的映射，并将其初始化备用。

```go
type Vertex struct {
	Lat, Long float64
}

var m map[string]Vertex // 方括号中为key的类型，方括号后面为value的类型

func main() {
	m = make(map[string]Vertex)
	m["Bell Labs"] = Vertex{
		40.68433, -74.39967,
	}
	fmt.Println(m["Bell Labs"])
}
```

### 映射的文法

映射的文法与结构体相似，不过必须有键名。

```go
var m = map[string]Vertex{
	"Bell Labs": Vertex{
		40.68433, -74.39967,
	},
	"Google": Vertex{
		37.42202, -122.08408,
	},
}
```

以上为映射初始化方法，key和value之间用冒号隔开，不同KV对之间用逗号隔开，最后一个也**必须有逗号**。

”Vertex“也可以省略。

### 修改映射

```go
m[key] = elem // 插入或修改元素
elem = m[key] // 获取元素
delete(m, key) // 删除元素
elem, ok = m[key] // 通过双赋值检测某个key是否存在
// 若key在m中，ok为true；否则，ok为false
// 若key不在映射中，那么elem是该映射元素类型的零值
elem, ok := m[key] // 若elem或ok还未声明，可以使用短变量声明
```

### 练习：映射

统计词频，保存到map中，可使用`strings.Fields`函数分割字符串。

```go
func WordCount(s string) map[string]int {
	m := make(map[string]int)
	words := strings.Fields(s)
	for _, word := range words {
		m[word]++
	}
	return m
}
```
### 函数值

函数也是值，可以作为函数的参数或返回值。也可以用`:=`声明一个函数变量。

```go
func compute(fn func(float64, float64) float64) float64 {
	return fn(3, 4)
}

func main() {
	hypot := func(x, y float64) float64 {
		return math.Sqrt(x*x + y*y)
	}
	fmt.Println(hypot(5, 12))

	fmt.Println(compute(hypot))
	fmt.Println(compute(math.Pow))
}
```

### 函数的闭包

Go 函数可以是一个闭包。闭包是一个函数值，它引用了其函数体之外的变量。该函数可以访问并赋予其引用的变量的值，换句话说，该函数被这些变量“绑定”在一起。

```go
func adder() func(int) int {
	sum := 0
	return func(x int) int {
		sum += x
		return sum
	}
}

func main() {
	pos, neg := adder(), adder()
	for i := 0; i < 10; i++ {
		fmt.Println(
			pos(i),
			neg(-2*i),
		)
	}
}
```

pos和neg各为一个闭包，都包含一个sum变量，初值为0。每次调用pos时，sum变量的值加i，然后输出sum。

可以把函数闭包理解成类的静态函数，sum为静态变量。

### 练习：斐波那契闭包

```go
// 返回一个“返回int的函数”
func fibonacci() func() int {
	a, b := 0, 1
	count := 0
	return func() int {
		count++
		if count == 1 {
			return 0
		} else if count == 2 {
			return 1
		} else {
			a, b = b, a + b
			return b
		}
	}
}

func main() {
	f := fibonacci()
	for i := 0; i < 10; i++ {
		fmt.Println(f())
	}
}
```

若count为1或2，直接返回第一项或第二项的值；否则，计算当前项的值，然后返回。

# Methods and interfaces

## 方法和接口

### 方法

Go没有类，但是可以为结构体定义方法。方法是一类带特殊的**接收者**参数的函数。

例如，为Vertex定义方法：
```go
func (v Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

func main() {
	v := Vertex{3, 4}
	fmt.Println(v.Abs())
}
```

也可为基本类型定义方法，不过要先定义类型别名，例如：`type MyFloat float64`，然后就可以为`MyFloat`类型定义方法。

### 指针接收者

注意，上述接收者是值接收者，只能用于访问；如果要修改结构体的成员，需要使用指针接收者：
`func (v *Vertex) Scale(f float64) {}`

### 方法与指针重定向

调用指针接收者方法时，可以通过变量调用，也可以通过指针调用：

```go
var v Vertex
v.Scale(5)  // OK
p := &v
p.Scale(10) // OK
```

### 选择值或指针作为接收者

使用指针接收者的原因有二：

- 首先，方法能够修改其接收者指向的值。

- 其次，这样可以避免在每次调用方法时复制该值。若值的类型为大型结构体时，这样做会更加高效。

通常来说，所有给定类型的方法都应该有值或指针接收者，但并不应该二者混用。

### 接口

接口类型是由一组方法签名定义的集合，接口类型的变量可以保存任何实现了这些方法的值。

```go
// 定义接口
type Abser interface {
	Abs() float64
}

// 实现Abs
func (f MyFloat) Abs() float64 {...} // 接收者为值
func (v *Vertex) Abs() float64 {...} // 接收者为指针

var a Abser // 定义接口变量
f := MyFloat(-math.Sqrt2)
v := Vertex{3, 4}
a = f  // a MyFloat 实现了 Abser
a = &v // a *Vertex 实现了 Abser
a = v // a Vertex 未实现 Abser，错误！
```

此处要注意，当只有指针实现了方法时，接口变量也只能用指针赋值。

### 接口与隐式实现

当一个类型实现了接口的所有方法时，它就实现了该接口，无需显式声明即可将该类型的变量直接赋值给接口变量。

> 隐式接口从接口的实现中解耦了定义，这样接口的实现可以出现在任何包中，无需提前准备。
>
> 因此，也就无需在每一个实现上增加新的接口名称，这样同时也鼓励了明确的接口定义。

### 接口值

接口也是值，可作为函数的参数或返回值。

在内部，接口值可以看作包含值和具体类型的元组：`(value, type)`。接口值保存了一个具体底层类型的具体值。接口值调用方法时会执行其底层类型的同名方法。

### 底层值为nil的接口值

即使接口内的具体值为nil，方法仍然会被nil接收者调用。

我们可以在方法中先判断接受者是否为空：

```go
func (t *T) M() {
	if t == nil {
		fmt.Println("<nil>")
		return
	}
	fmt.Println(t.S)
}
```

执行下列代码时，控制台打印<nil>：

```go
var i I
var t *T
i = t
i.M()
```

### nil接口值

nil 接口值既不保存值也不保存具体类型。

为 nil 接口调用方法会产生运行时错误，因为接口的元组内并未包含能够指明该调用哪个具体方法的类型。

### 空接口

指定了0个方法的接口为空接口：

`interface {}`

空接口可以保存任何类型的值，因为任何类型的值都至少实现了0个接口。

空接口被用来处理未知类型的值。

```go
func main() {
	var i interface{}
	describe(i)

	i = 42
	describe(i)

	i = "hello"
	describe(i)
}

func describe(i interface{}) {
	fmt.Printf("(%v, %T)\n", i, i)
}
```

### 类型断言

类型断言提供了访问接口值底层具体值的方式。类型断言的语法与判断map中是否存在某个key非常相似。

```go
t := i.(T) // 若接口i保存了类型T，则将其赋值给t，否则触发一个panic
t, ok := i.(T) // ok为false时，t为零值，不会触发panic
```

### 类型选择

类型选择使用switch-case语句，从几个类型断言中选择分支：

```go
switch v := i.(type) {
case T:
case S:
default:
}
```

其声明语法与类型断言类似，不过具体类型值换成了`type`。**当匹配成功时，变量v会被赋值为相应类型的值（是具体类型，不是接口）**。

### Stringer

```go
type Stringer interface {
    String() string
}
```

Stringer接口中的String函数规定了变量被fmt.Println函数打印的格式。

### 练习：Stringer

为IP地址实现Stringer接口。注意，IP地址是用byte类型存储，首先要将其转换成int，再将int使用`strconv.Itoa`转换成string，不能一步到位直接byte转string，因为byte中存储的不是ASCII码。

```go
package main

import (
	"fmt"
	"strconv"
)

type IPAddr [4]byte

// TODO: 给 IPAddr 添加一个 "String() string" 方法
func (ip IPAddr) String() string {
	ip_str := ""
	for i := 0; i < 4; i++ {
		ip_str += strconv.Itoa(int(ip[i]))
		if i != 3 {
			ip_str += "."
		}
	}
	
	return ip_str
}

func main() {
	hosts := map[string]IPAddr{
		"loopback":  {127, 0, 0, 1},
		"googleDNS": {8, 8, 8, 8},
	}
	for name, ip := range hosts {
		fmt.Printf("%v: %v\n", name, ip)
	}
}
```

### 错误

错误error也是一个接口。同Stringer一样，Error函数返回的字符串也用于fmt包打印时。

```go
type error interface {
    Error() string
}
```

通常函数会返回一个error值，应判断其是否为nil：

```go
i, err := strconv.Atoi("42")
```

### 练习：错误

修改之前的Sqrt函数，当输入值为负时返回一个error。

```go
package main

import (
	"fmt"
)

type ErrNegativeSqrt float64 // 将float64定义为错误类型

func (e ErrNegativeSqrt) Error() string {
	// ErrNegativeSqrt类型实现了error接口，因此它是一个error类型
	str := "cannot Sqrt negative number: " + fmt.Sprintf("%f", float64(e)) // 注意Sprintf函数的使用
	return str
}

func Sqrt(x float64) (float64, error) {
	if x < 0 {
		var err ErrNegativeSqrt = ErrNegativeSqrt(x) // 创建一个错误类型变量，将x值赋值给它
		return x, err
	}
	
	z := 1.0
	times := 10
	for count := 0; count < times; count++ {
		z -= (z * z - x) / (2 * z)
	}
	return z, nil
}

func main() {
	fmt.Println(Sqrt(2))
	fmt.Println(Sqrt(-2))
}
```

### Reader

io.Reader接口中有一个Read方法：
`func (T) Read(b []byte) (n int, err error)`

用于从类型T的变量读取前n个字节，存放到切片b中。在遇到输入流的结尾时，它会返回一个io.EOF错误。

### 练习：Reader

实现一个无限字符流'A'。可以通过len获取切片长度，然后将其填满。

```go
// TODO: 给 MyReader 添加一个 Read([]byte) (int, error) 方法
func (r MyReader) Read(slice []byte) (int, error) {
	for i := 0; i < len(slice); i++ {
		slice[i] = 'A'
	}
	return len(slice), nil
}
```

### 练习：rot13Reader

```go
func (rd rot13Reader) Read(slice []byte) (n int, err error) {
	n, err = rd.r.Read(slice)
	for i := 0; i < n; i++ {
		slice[i] = rot13(slice[i])
	}
	return
}
```

### 图像

```go
package image

type Image interface {
    ColorModel() color.Model
    Bounds() Rectangle // 图像边界
    At(x, y int) color.Color // 访问某个像素点
}
```

### 练习：图像

使用image接口实现图像。

```go
package main

import (
	"golang.org/x/tour/pic"
	"image/color"
	"image"
)

type Image struct{}

func (img Image) ColorModel() color.Model {
	return color.RGBAModel
}

func (img Image) Bounds() image.Rectangle {
	return image.Rect(0, 0, 800, 600)
}

func (img Image) At(x, y int) color.Color {
	return color.RGBA{(uint8)(x*y), (uint8)(x*y), 255, 255}
}

func main() {
	m := Image{}
	pic.ShowImage(m)
}
```

# Concurrency

## 并发

### Go程

Go程（goroutine）是由 Go 运行时管理的轻量级线程。

`go f(x, y, z)`创建一个新的goroutine并执行f函数。

goroutine之间共享内存。

### 信道

```go
ch := make(chan int) // 创建信道
ch <- v    // 将 v 发送至信道 ch。
v := <-ch  // 从 ch 接收值并赋予 v。
```

默认情况下，信道的发送和接受是阻塞的。

```go
func sum(s []int, c chan int) {
	sum := 0
	for _, v := range s {
		sum += v
	}
	c <- sum // 将和送入 c
}

func main() {
	s := []int{7, 2, 8, -9, 4, 0}

	c := make(chan int)
	go sum(s[:len(s)/2], c)
	go sum(s[len(s)/2:], c)
	x, y := <-c, <-c // 从 c 中接收

	fmt.Println(x, y, x+y)
}
```

注意，两次`<-c`并不代表信道可以存放多个值。创建两个goroutine之后，主线程在接受处阻塞。每当一个goroutine完成计算，向信道写入值时，主进程立即读出，此时另外一个goroutine才能再进行写入。

### 带缓冲的信道

`ch := make(chan int, 100)`创建了一个大小为100的信道。仅当缓冲区填满时，向其发送数据才会阻塞。当缓冲区为空时，接收方会阻塞。

### range和close

发送者可以通过close关闭信道，表示没有需要发送的值了。接收者可以通过第二个参数测试信道是否被关闭：`v, ok := <-ch`

循环`for i := range c`会从信道不断接收值，直到它被关闭。

只有发送者能关闭信道，接收者不能。向一个已经关闭的信道发送数据会引发panic。

**注意：信道与文件不同，通常情况下无需关闭它们。只有在必须告诉接收者不再有需要发送的值时才有必要关闭，例如终止一个 range 循环。**

### select语句

select 语句使一个 Go 程可以等待多个通信操作。

select 会阻塞到某个分支可以继续执行为止，这时就会执行该分支。当多个分支都准备好时会随机选择一个执行。

```go
func fibonacci(c, quit chan int) {
	x, y := 0, 1
	for {
		select {
		case c <- x:
			x, y = y, x+y
		case <-quit:
			fmt.Println("quit")
			return
		}
	}
}

func main() {
	c := make(chan int)
	quit := make(chan int)
	go func() {
		for i := 0; i < 10; i++ {
			fmt.Println(<-c)
		}
		quit <- 0
	}() // 学习此处的匿名函数写法
	fibonacci(c, quit)
}
```

### 默认选择

当select中的其他分支都没有准备好时，执行default分支。用于避免程序阻塞。

### 练习：等价二叉查找树

```go
package main

import (
	"fmt"
	"golang.org/x/tour/tree"
)

// Walk 步进 tree t 将所有的值从 tree 发送到 channel ch。
func Walk(t *tree.Tree, ch chan int) {
	if (t != nil) {
		Walk(t.Left, ch)
		ch <- t.Value
		Walk(t.Right, ch)
	}
}

// Same 检测树 t1 和 t2 是否含有相同的值。
func Same(t1, t2 *tree.Tree) bool {
	ch1, ch2 := make(chan int), make(chan int)
	go Walk(t1, ch1)
	go Walk(t2, ch2)
	for i := 0; i < 10; i++ {
		v1 := <-ch1
		v2 := <-ch2
		if v1 != v2 {
			return false
		}
	}
	return true
}

func main() {
	t1 := tree.New(1)
	t2 := tree.New(1)
	fmt.Println(Same(t1, t2))
}
```

### sync.Mutex

Mutex变量可进行Lock和Unlock。可使用defer保证互斥锁一定会被解锁。学习下列例子中defer的用法：

```go
// Value 返回给定 key 的计数器的当前值。
func (c *SafeCounter) Value(key string) int {
	c.mux.Lock()
	// Lock 之后同一时刻只有一个 goroutine 能访问 c.v
	defer c.mux.Unlock()
	return c.v[key]
}
```

### 练习：Web爬虫

```go
package main

import (
	"fmt"
	"sync"
)

type Fetcher interface {
	// Fetch 返回 URL 的 body 内容，并且将在这个页面上找到的 URL 放到一个 slice 中。
	Fetch(url string) (body string, urls []string, err error)
}

var crawled map[string]bool = make(map[string]bool)
var mutex sync.Mutex

// Crawl 使用 fetcher 从某个 URL 开始递归的爬取页面，直到达到最大深度。
func Crawl(url string, depth int, fetcher Fetcher, ch chan bool) {
	// TODO: 并行的抓取 URL。
	// TODO: 不重复抓取页面。
	// fmt.Println(url, depth)
	if depth <= 0 {
		ch <- true
		return
	}
	body, urls, err := fetcher.Fetch(url)
	if err != nil {
		fmt.Println(err)
		ch <- true
		return
	}
	
	mutex.Lock()
	if (crawled[url] == false) {
		crawled[url] = true
		mutex.Unlock()
		fmt.Printf("found: %s %q\n", url, body)
		
		subCh := make(chan bool)
		for _, u := range urls {
			go Crawl(u, depth-1, fetcher, subCh)
		}
		for i := 0; i < len(urls); i++ {
			<- subCh
		}
	} else {
		mutex.Unlock()
	}
	
	ch <- true
	return
}

func main() {
	ch := make(chan bool)
	go Crawl("https://golang.org/", 4, fetcher, ch)
	<- ch
}

// fakeFetcher 是返回若干结果的 Fetcher。
type fakeFetcher map[string]*fakeResult

type fakeResult struct {
	body string
	urls []string
}

func (f fakeFetcher) Fetch(url string) (string, []string, error) {
	if res, ok := f[url]; ok {
		return res.body, res.urls, nil
	}
	return "", nil, fmt.Errorf("not found: %s", url)
}

// fetcher 是填充后的 fakeFetcher。
var fetcher = fakeFetcher{
	"https://golang.org/": &fakeResult{
		"The Go Programming Language",
		[]string{
			"https://golang.org/pkg/",
			"https://golang.org/cmd/",
		},
	},
	"https://golang.org/pkg/": &fakeResult{
		"Packages",
		[]string{
			"https://golang.org/",
			"https://golang.org/cmd/",
			"https://golang.org/pkg/fmt/",
			"https://golang.org/pkg/os/",
		},
	},
	"https://golang.org/pkg/fmt/": &fakeResult{
		"Package fmt",
		[]string{
			"https://golang.org/",
			"https://golang.org/pkg/",
		},
	},
	"https://golang.org/pkg/os/": &fakeResult{
		"Package os",
		[]string{
			"https://golang.org/",
			"https://golang.org/pkg/",
		},
	},
}
```

信道的作用在于等待所有goroutine完成后再退出本线程，相当于pthread_join。