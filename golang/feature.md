### Go语言参数传递是值传递还是引用传递？

先说结论: 值传递。但却需要区分传递的参数是引用类型还是非引用类型，以及传递的参数有没有指针（包括传递的参数子元素是否包含指针）[参考](https://www.flysnow.org/2018/02/24/golang-function-parameters-passed-by-value.html)

* 引用类型：`slice`、`map`、`channel`
* 引用类型不是传引用
* `make`的`map`和`channel`本质上是指针类型, `slice`不是（如`map`用 `make`函数返回的是一个`hmap`类型的指针`*hmap`，`slice`本质上是结构体，`make`函数总是返回初始化后类型变量本身）, [参考slice特性](/golang/slice.md)
* 自定义的Struct不是引用类型, 但若其子元素有指针类型时, 其特性将类似于slice

### 以显式的指针传递为例

```go
package main
import "log"
func test(s *struct {t int}) *struct {t int} {
   // 0xc000096000 0xc00008e020
   // 按%p输出 s，表示输出指针s的值，也即指针s指向的变量内存地址
   // 按%p输出 &s，表示输出指针变量s的内存地址
   // 需要理解一下，指针的值区别于指针变量的地址
   // 指针的值是它指向的变量地址，但它本身也是一个变量，所以它自己也有一个内存地址
   // 可以看出，在外部的指针变量地址与函数内部的指针变量地址不一致
   // 指针的值是一致的，因此函数参数的传递是值传递，发生了值拷贝
   log.Printf("%p %p\n", s, &s)
   return s
}
func main() {
   s := &struct {
      t int
   }{1}
   // 0xc000096000 0xc00008e010
   log.Printf("%p %p\n", s, &s)
   test(s)
}
```

#### 以slice传递为案例

```go
package main
import "log"
func modifySli(sli []int) {
	// 0xc00000c060
	//&sli已经是新的了, 说明函数是值传递(若为引用传递, 需要与调用时的地址保持一致)
	log.Printf("slice address in modifySli:%p\n", &sli)
	sli[0] = 2
}
func modifySli1(sli []int) {
	// 由于传进来的参数，cap为1，已经有了1个元素，append之后发生扩容
	sli = append(sli, 2)
	sli[0] = 1
}
func main() {
	s := make([]int, 0, 1)
	s = append(s, 1)
	// 0xc00000c040 [1]
	log.Printf("%p %+v\n", &s, s)
	modifySli(s)
	// 0xc00000c040 [2]
	// 函数调用没有传递指针, modifySli函数也成功修改了slice的数据
	log.Printf("%p %+v\n", &s, s)
	modifySli1(s)
	// 0xc00000c040 [2]
	// 在 modifySli1 中，sli发生了扩容，在该函数中sli变量拥有了新的底层数组，就影响不到外面的s了
	log.Printf("%p %+v\n", &s, s)
}
```



