# 了解一下Go中的"sb"代码？


<!--more-->

## 一、前言
相信很多`java`程序员都背过或者用过`StringBuilder`或者`StringBuffer`这种`sb`代码。
最近工作需要查看`Helm`的源码的时候，注意到一个特别使用的字符串连接的方法`strings.Builder`，咋一看特别像`java`的`StringBuilder`，所以研究了下它，以及了解了下其他连接字符串的方法，结果发现确实`strings.Builder`的效果显著。
## 二、内容
### 1.首先我放出我了解的几个方法，并给出在我本地电脑跑`benchmark`的效果，[点击查看代码](https://play.studygolang.com/p/EC8uZQAIFsN)：
#### ①.传统的`+`号连接
```go
func BenchmarkTestStrPlus(b *testing.B) {
	var result string
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		result = strconv.Itoa(i) + result
	}
}
```
#### ②.`fmt.Sprintf`连接
```go
func BenchmarkTestStrSprintf(b *testing.B) {
	var result string
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		result = fmt.Sprintf("%s%s", result, strconv.Itoa(i))
	}
}
```
#### ③. `strings.Join`连接
```go
func BenchmarkTestJoin(b *testing.B) {
	var result string
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		result = strings.Join([]string{result, strconv.Itoa(i)}, "")
	}
}
```
#### ④.`bytes.Buffer`连接
```go
func BenchmarkTestBuffer(b *testing.B) {
	var result bytes.Buffer
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		if _, err := result.WriteString(strconv.Itoa(i)); err != nil {
			panic(err)
		}
	}
}
```
#### ⑤.最后就是`strings.Builder`方法连接
```go
func BenchmarkTestBuilder(b *testing.B) {
	var result strings.Builder
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		result.WriteString(strconv.Itoa(i))
	}
}
```
以上就是我了解到的5种字符串拼接的方法，大家可以猜测下他们的效率顺序，大家如果谁对我这种测试方法质疑的，可以留言讨论。csdn链接:[https://blog.csdn.net/u010927340/article/details/118120669](https://blog.csdn.net/u010927340/article/details/118120669)

我觉得第二种性能应该最差，毕竟它支持的各种类型太多，一般来说兼容性是以性能降低的代价。其次就是第一钟，最原始的拼接的字符串的方式，为什么呢？这个和`java`的`String`很像，都是`immutable`，换言之就是当更改这个`string`对象的时候，其实并没有更改该`string`，而是新增了一个`string`，但是给人的感觉好像把它修改了，为什么这么设计，大家可以自行百度，但是该设计会导致在拼接字符串的时候会产生大量的`string`，不仅耗时，还耗内存，更有甚者导致`STW`。然后我觉得会是第三种，点开看了下他的`Join`方法，竟然里面使用了`strings.Builder`，可惜他是直接返回了`string`，相当于每个`Join`操作也产生了新的`string`，所以我把她放到第三位。最后就是第4和第5的PK，还是我上面提到的判断思路：谁更专业肯定效率最好。所以我觉得第5种性能还是要比第4种好。

到此我的运行前的性能判断，从高到低如下:
```shell
5 < 4 < 3 < 1 < 2
```
好的用我的渣渣电脑运行，结果如下（事实上我运行了很多次，性能指标大小顺序都是一致的）：
```shell
cpu: Intel(R) Core(TM) i5-5257U CPU @ 2.70GHz
BenchmarkTestStrPlus
BenchmarkTestStrPlus-4      	  187838	    226381 ns/op
BenchmarkTestStrSprintf
BenchmarkTestStrSprintf-4   	  130268	    306116 ns/op
BenchmarkTestJoin
BenchmarkTestJoin-4         	  182164	    289074 ns/op
BenchmarkTestBuffer
BenchmarkTestBuffer-4       	20228474	       107.1 ns/op
BenchmarkTestBuilder
BenchmarkTestBuilder-4      	10084845	        99.34 ns/op
```
事实上的性能从高到低的结果如下:
```shell
5 < 4 < 1 < 3 < 2
```
大型翻车现场，原来原始的方式竟然没想象中那么差，`string`的原始拼接在很多语言中都分别在编译期和运行时都有特地优化过，毕竟它的使用频率非常高，优化它就相当于优化了整个语言。这个估计能查资料才能找到具体原因，在这里我不关心到底为什么性能比3还强，我这里只聚焦于`strings.Builder`，因为从宏观上来说他肯定比除第4种方法外都强，在这里也不额外关心为什么`bytes.Buffer`以微弱的劣势输于`strings.Builder`，事实上我发现`strings.Builder`的实现和`bytes.Buffer`的原理很像，都是操作`byte`数组，但是`bytes.Buffer`的功能更强。我关心的是这个结论很重要。下面我们一起领略下`strings.Builder`的设计吧。

### 2.`strings.Builder`的源码很简单，加上注释也才100多行
我给大家列一下核心代码，那就更少了
```go
// A Builder is used to efficiently build a string using Write methods.
// It minimizes memory copying. The zero value is ready to use.
// Do not copy a non-zero Builder.
type Builder struct {
	addr *Builder // of receiver, to detect copies by value
	buf  []byte
}

func (b *Builder) copyCheck() {
	if b.addr == nil {
		b.addr = (*Builder)(noescape(unsafe.Pointer(b)))
	} else if b.addr != b {
		panic("strings: illegal use of non-zero Builder copied by value")
	}
}

func (b *Builder) String() string {
	return *(*string)(unsafe.Pointer(&b.buf))
}

// grow copies the buffer to a new, larger buffer so that there are at least n bytes of capacity beyond len(b.buf).
func (b *Builder) grow(n int) {
	buf := make([]byte, len(b.buf), 2*cap(b.buf)+n)
	copy(buf, b.buf)
	b.buf = buf
}

// Grow grows b's capacity, if necessary, to guarantee space for another n bytes. After Grow(n), at least n bytes can be written to b without another allocation. If n is negative, Grow panics.
func (b *Builder) Grow(n int) {
	b.copyCheck()
	if n < 0 {
		panic("strings.Builder.Grow: negative count")
	}
	if cap(b.buf)-len(b.buf) < n {
		b.grow(n)
	}
}

func (b *Builder) Write(p []byte) (int, error) {
	b.copyCheck()
	b.buf = append(b.buf, p...)
	return len(p), nil
}

func (b *Builder) WriteByte(c byte) error {
	b.copyCheck()
	b.buf = append(b.buf, c)
	return nil
}

func (b *Builder) WriteRune(r rune) (int, error) {
	b.copyCheck()
	if r < utf8.RuneSelf {
		b.buf = append(b.buf, byte(r))
		return 1, nil
	}
	l := len(b.buf)
	if cap(b.buf)-l < utf8.UTFMax {
		b.grow(utf8.UTFMax)
	}
	n := utf8.EncodeRune(b.buf[l:l+utf8.UTFMax], r)
	b.buf = b.buf[:l+n]
	return n, nil
}

func (b *Builder) WriteString(s string) (int, error) {
	b.copyCheck()
	b.buf = append(b.buf, s...)
	return len(s), nil
}

// Reset resets the Builder to be empty.
func (b *Builder) Reset() {
	b.addr = nil
	b.buf = nil
}
```

首先实现了`io`包中的`Writer``StringWriter``ByteWriter`接口，也就是说在其他地方接受这3个接口的地方都能用`strings.Builder`替代写入。

开头`1~4`行的注释就申明了`Builder`的设计宗旨：`尽可能避免内存拷贝`，而且还特地提醒了：`Builder不能被拷贝`。为什么不能被拷贝呢？`Builder`的设计宗旨就是避免内存拷贝，但是如果说再拷贝Builder的话就违背了。

为了做到上面2点，`Builder`的结构体就体现了:
```go
type Builder struct {
	addr *Builder // of receiver, to detect copies by value
	buf  []byte
}
```
里面包括一个字节数组，在go中`string`底层其实就是字节数组，为什么这里也用了数组呢？底层字节数组方便往数组里面塞东西，自己可以控制扩容，扩多少。这样不会像`string`的原始拼接方式那样产生大量的`string`对象。还有1个指向自己的`Builder`指针，正如注释所说：检测是否有拷贝了整个`Builder`对象。如果拷贝了会怎样呢？见`copyCheck`代码：
```go
func (b *Builder) copyCheck() {
	if b.addr == nil {
		// This hack works around a failing of Go's escape analysis
		// that was causing b to escape and be heap allocated.
		// See issue 23382.
		// TODO: once issue 7921 is fixed, this should be reverted to
		// just "b.addr = b".
		b.addr = (*Builder)(noescape(unsafe.Pointer(b)))
	} else if b.addr != b {
		panic("strings: illegal use of non-zero Builder copied by value")
	}
}
```
会直接发生`panic`，很霸道，直接被`Builder`拒绝了。
接下来我写一个拷贝`Builder`的[代码](https://play.studygolang.com/p/bDt9lGImCiy)验证下，代码链接[https://play.studygolang.com/p/bDt9lGImCiy](https://play.studygolang.com/p/bDt9lGImCiy)：

```go
var sb strings.Builder
sb.WriteString("1")
var sb_copy = *(*strings.Builder)(unsafe.Pointer(&sb))
sb_copy.WriteString("2")
```
如下为`panic`的输出:
```shell
panic: strings: illegal use of non-zero Builder copied by value

goroutine 1 [running]:
strings.(*Builder).copyCheck(...)
	/usr/local/go-faketime/src/strings/builder.go:42
strings.(*Builder).WriteString(...)
	/usr/local/go-faketime/src/strings/builder.go:122
main.main()
	/tmp/sandbox194128443/prog.go:12 +0x1ad
```
所有`Write*`操作都有`copyCheck`的拷贝检测，如果有发生类似的拷贝，都会`panic`。但是它强调了`Builder`的重用功能，需要`Reset`之后才能构造下一个`String`。

接下来要说明的是`Grow`方法，该方法是提供的一个`public`的方法用来给`Builder`扩容`buf`的数组。为什么要提供这个呢？可以看到除了`WriteRune`必要的时候主动扩容了，其他`Write*`方法并没有去事先判断扩容的-依赖切片的自动判断扩容。其实`Builder`提供这个`Grow`方法是给使用者使用，提前能判断好长度，然后扩容，这样后期调用`Write*`方法就没有内存拷贝的操作，因为一旦发生扩容的话就会有2个问题：分配内存和拷贝。

代码34行判断如果`buf`数组的剩余容量小于需要扩容的长度，那么才去调用私有的`grow`去扩容:
```go
// grow copies the buffer to a new, larger buffer so that there are at least n
// bytes of capacity beyond len(b.buf).
func (b *Builder) grow(n int) {
	buf := make([]byte, len(b.buf), 2*cap(b.buf)+n)
	copy(buf, b.buf)
	b.buf = buf
}
```
这几行代码很清晰，就是分配内存，容量是以前的容量再加上需要扩容的数量，这样就确保长度`n`的字节能存到`Builder`去。

接下来看看`WriteRune`方法，因为其他的`Write*`方法太简单，就略过了。
```go
func (b *Builder) WriteRune(r rune) (int, error) {
	b.copyCheck()
	if r < utf8.RuneSelf {
		b.buf = append(b.buf, byte(r))
		return 1, nil
	}
	l := len(b.buf)
	if cap(b.buf)-l < utf8.UTFMax {
		b.grow(utf8.UTFMax)
	}
	n := utf8.EncodeRune(b.buf[l:l+utf8.UTFMax], r)
	b.buf = b.buf[:l+n]
	return n, nil
}
```
`rune`在`go`中是一个特殊类型，用来表示一个`utf-8`的一个字符，`utf-8`是一个动态字节，最长有4个字节，细究发现`rune`其实是`int32`的别名，`int32`正好是4个字节，所以刚刚好。也就是`WriteRune`方法是用来写入`utf-8`的字符。第3行表示这个字符是否是1个字节，如果可以就直接追加到切片。`RuneSelf`表示就是一个`utf-8`字符最大的单字节大小。

第8行判断剩余的容量是否大于`utf-8`的最大字节`UTFMax`，也就是4，如果小于则能提前扩容4个字节长度。然后11行开始把`rune`字符的字节依次复制到`buf`切片里面去。返回的结果n表示复制了多少个字节到切片里面去，因为`rune`是1-4个字节。

注意第12行的操作，如果debug下会发现，在执行11行代码之后，`buf`并没有变化，这是因为11行的入参`b.buf[l:l+utf8.UTFMax]`其实是产生了一个新的切片，切片的结构如下:
```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```
`array`是存储切片的真正的值，虽然说2个切片都维护同一个`array`，但是`len`和`cap`其实是切片自己维护的，所以不执行12行的操作的时候，结果就是执行`len`和`cap`的结果没有变化。执行之后`len`和`cap`就发生变化，且能打印出来整个`array`的值。

当写完了所有的字符串的字符之后，要得到拼接后的字符串需要调用`String`方法:
```go
// String returns the accumulated string.
func (b *Builder) String() string {
	return *(*string)(unsafe.Pointer(&b.buf))
}
```
`b.buf`是个切片，而上面分析了切片的底层其实就是`array`数组，`string`的结构体如下:
```go
type stringStruct struct {
	str unsafe.Pointer
	len int
}
```
和上面的slice的结构体相似，故能直接把切片`b.buf`转为`*string`。
## 三、总结
上面分析了`strings.Builder`的源码，知道了拼接字符串的底层逻辑，所以如果有大量的`string`对象需要拼接，那么`strings.Builder`非常合适，而且最好知道所有要拼接的`string`的长度总和，事先分配好内存，还能进一步提高效率。

而且我发现其实`Reset`方法还可以优化的角度，不用把`b.buf`设为`nil`，这样的话以前申请的`buf`的数组就会被回收掉，我觉得可以利用起来，不用下次拼接字符串的时候再申请内存。不知道你怎么看这个问题呢？

- 我的微信公众号:
![云原生玩码部落](https://img-blog.csdnimg.cn/20201213215616541.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA5MjczNDA=,size_16,color_FFFFFF,t_70)

