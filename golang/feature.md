### Go语言参数传递是值传递还是引用传递？

先说结论: 值传递。但却需要区分传递的参数是引用类型还是非引用类型，以及传递的参数有没有指针（包括传递的参数子元素是否包含指针）

* 引用类型：slice、map、channel
* 引用类型不是传引用
* make的map和channel是指针类型, slice不是, [参考slice特性](/golang/slice.md)
* 自定义的Struct不是引用类型, 但若其子元素有指针类型时, 其特性将类似于slice

#### 以slice为案例

```go
package main
import "fmt"
func modifySli(sli []int) {
  // 0xc00000c060
  //&sli已经是新的了, 说明函数是值传递(若为引用传递, 需要与调用时的地址保持一致)
	fmt.Printf("slice address in modifySli:%p\n", &sli) 
	sli[0] = 2
}
func main() {
	s := make([]int, 0, 1)
	s = append(s, 1)
  // 0xc00000c040 [1]
	fmt.Printf("%p %+v\n", &s, s)  
	modifySli(s)
  // 0xc00000c040 [2] 
  // 函数调用没有传递指针, modifySli函数也成功修改了slice的数据
	fmt.Printf("%p %+v\n", &s, s)  
}
```

[参考](https://www.flysnow.org/2018/02/24/golang-function-parameters-passed-by-value.html)



