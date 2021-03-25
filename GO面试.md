# GO面试

## 字符串拼接性能分析

- 字符串(string) 是不可变的，拼接字符串事实上是创建了一个新的字符串对象。

#### 拼接方式

- +，性能差
- fmt.Sprintf，性能差
- strings.Builder，性能好
- bytes.Buffer，性能好
- []byte，性能好

#### 使用strings.Builder构造字符串

```go
func builderConcat(n int, str string) string {
	var builder strings.Builder
	builder.Grow(n * len(str)) // 预申请内存，比使用byte[]的方式还要少一次 byte转string 的内存申请
	for i := 0; i < n; i++ {
		builder.WriteString(str)
	}
	return builder.String()
}
```

## 切片性能分析

- 在已有切片的基础上进行切片，不会创建新的底层数组。因为原来的底层数组没有发生变化，内存会一直占用，直到没有变量引用该数组。
- 当原切片由大量的元素构成，但是我们在原切片的基础上切片，虽然只使用了很小一段，但底层数组在内存中仍然占据了大量空间，得不到释放。
- 使用 `copy` 替代 `re-slice`

## for和range性能比较

- 当for和range (只迭代下标时) 性能上无差异，如遍历 [ ]struct{}、[ ]int等
- range 对每个迭代值都创建了一个拷贝。因此如果每次迭代的值内存占用很小的情况下，for 和 range 的性能几乎没有差异，但是如果每个迭代值内存占用很大
- 使用 range 同时迭代下标和值，则需要将切片/数组的元素改为指针；使用 range，建议只迭代下标，通过下标访问迭代值

## 反射性能

- 对于一个普通的拥有 4 个字段的结构体 `Config` 来说，使用反射给每个字段赋值，相比直接赋值，性能劣化约 100 - 1000 倍。其中，`FieldByName` 的性能相比 `Field` 劣化 10 倍

```go
func BenchmarkSet(b *testing.B) {
	config := new(Config)
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		config.Name = "name"
		config.IP = "ip"
		config.URL = "url"
		config.Timeout = "timeout"
	}
}

func BenchmarkReflect_FieldSet(b *testing.B) {
	typ := reflect.TypeOf(Config{})
	ins := reflect.New(typ).Elem()
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		ins.Field(0).SetString("name")
		ins.Field(1).SetString("ip")
		ins.Field(2).SetString("url")
		ins.Field(3).SetString("timeout")
	}
}

func BenchmarkReflect_FieldByNameSet(b *testing.B) {
	typ := reflect.TypeOf(Config{})
	ins := reflect.New(typ).Elem()
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		ins.FieldByName("Name").SetString("name")
		ins.FieldByName("IP").SetString("ip")
		ins.FieldByName("URL").SetString("url")
		ins.FieldByName("Timeout").SetString("timeout")
	}
}
```

```shell
$ go test -bench="Set$" .          
goos: darwin
goarch: amd64
pkg: example/hpg-reflect
BenchmarkSet-8                          1000000000               0.302 ns/op
BenchmarkReflect_FieldSet-8             33913672                34.5 ns/op
BenchmarkReflect_FieldByNameSet-8        3775234               316 ns/op
PASS
ok      example/hpg-reflect     3.066s
```

- `FieldByName`调用链

  (v Value) FieldByName -> (t *rtype) FieldByName -> (t *structType) FieldByName

-  `(t *structType) FieldByName` 中使用 for 循环，逐个字段查找，字段名匹配时返回。也就是说，在反射的内部，字段是按顺序存储的，因此按照下标访问查询效率为 O(1)，而按照 `Name` 访问，则需要遍历所有字段，查询效率为 O(N)。结构体所包含的字段(包括方法)越多，那么两者之间的效率差距则越大。

#### 性能的提高

- 在序列化和反序列化时尽量避免使用反射，使用提供的json库
- 缓存，利用map将结构体中的Name和Index保存起来



## 空结构体

- 大小：unsafe.Sizeof(struct{}{}) ==> 0
- 空结构体不占据内存空间，被广泛作为各种占位符使用
  - 实现集合。将 map 作为集合(Set)使用时，可以将值类型定义为空结构体，仅作为占位符
  - 不发送数据的channel
  - 仅包含方法的结构体

#### 结构体对齐

- 合理的内存对齐可以提高内存读写的性能，并且便于实现变量操作的原子性。
- `unsafe.Alignof(Args{})`可以返回内存对齐方式

#### 内存对齐技巧

- #### 合理布局减少内存占用	

  - ###### 调整结构体中字段的顺序以减少内存占用

- #### 空 struct{} 的对齐      

  - ###### 当 `struct{}` 作为其他 struct 最后一个字段时，需要填充额外的内存保证安全。

  - ###### 因为如果有指针指向该字段, 返回的地址将在结构体之外，如果此指针一直存活不释放对应的内存，就会有内存泄露

## 读写锁和互斥锁的比较

- 读写比为 9:1 时，读写锁的性能约为互斥锁的 8 倍
- 读写比为 1:9 时，读写锁性能相当
- 读写比为 5:5 时，读写锁的性能约为互斥锁的 2 倍

## 协程泄露

- 协程泄露是指协程创建后，长时间得不到释放(阻塞)，并且还在不断地创建新的协程，最终导致内存耗尽，程序崩溃。常见的导致协程泄露的场景有以下几种：
  - 缺少收发器，导致发，收阻塞
  - 死锁
  - 无限循环

## GOMAXPROCS` 或 `runtime.GOMAXPROCS(num int)

- `GOMAXPROCS` 限制的是同时执行用户态 Go 代码的操作系统线程的数量，但是对于被系统调用阻塞的线程数量是没有限制的。
- `GOMAXPROCS` 的默认值等于 CPU 的逻辑核数，同一时间，一个核只能绑定一个线程，然后运行被调度的协程。

## 常量

- Go 语言中，常量分为无类型常量和有类型常量两种，`const N = 100`，属于无类型常量，赋值给其他变量时，如果字面量能够转换为对应类型的变量，则赋值成功，例如，`var x int = N`。
- 对于有类型的常量 `const M int32 = 100`，赋值给其他变量时，需要类型匹配才能成功

```go
func main() {
	var a int8 = -1	  // int8 能表示的数字的范围是 [-2^7, 2^7-1]
  // 如果a定义为 const a int8 = -1 则发生编译错误，常量除以常量结果仍是常量，常量不允许溢出
	var b int8 = -128 / a		
  // -128 是无类型常量，转换为 int8，再除以变量 -1，结果为 128，常量除以变量，结果是一个变量
	fmt.Println(b)
  // 变量转换时允许溢出，符号位变为1，转为补码后恰好等于 -128
}
```

```basic
-1 :  11111111
00000001(原码)    11111110(取反)    11111111(加一)
-128：    
10000000(原码)    01111111(取反)    10000000(加一)

-1 + 1 = 0
11111111 + 00000001 = 00000000(最高位溢出省略)
-128 + 127 = -1
10000000 + 01111111 = 11111111
```

## defer

- defer 延迟调用时，需要保存函数指针和参数，因此**链式调用**的情况下，除了最后一个函数/方法外的函数/方法都会在调用时直接执行。

- ```
  defer t.f(1).f(2) //会直接调用f(1)再延迟调用f2
  ```