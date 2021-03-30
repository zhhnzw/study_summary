## 切片

```go
// slice 数据结构
type slice struct {
	array unsafe.Pointer 
	len   int            
	cap   int            
}
```

fmt特殊处理了slice类型, 以`%p`打印`slice`其实打印的是该slice变量对应的底层数组的首元素的地址

以`%p`打印`&slice`打印的是该`slice`(结构体)变量的地址

```go
func main() {
	array := [20]int{1, 2, 3, 4, 5, 6, 7, 8, 9}
	fmt.Printf("%p\n", &array) // 0xc000096000
	s1 := array[:5]
	s2 := append(s1, 10)
	fmt.Printf("%p s1 =%+v\n", s1, s1) // 0xc000096000 [1 2 3 4 5]
	fmt.Printf("%p s2 =%+v\n", s2, s2) // 0xc000096000 [1 2 3 4 5 10]
	s2[0] = 0
	fmt.Printf("%p s1 =%+v\n", s1, s1) // 0xc000096000 [0 2 3 4 5]
	fmt.Printf("%p s2 =%+v\n", s2, s2) // 0xc000096000 [0 2 3 4 5 10]
}
```

为什么取s2下标的操作能修改s1的值, 而`append`函数却没能改变s1变量的值？

答: s1和s2确实是指向同一个底层数组，只是slice变量对应的len不同。

注: `s1 := array[:5]`与`s1 := array[5:]`对应的底层数组地址不一样, 但其本质还是指向了同一个底层数组，当修改公共元素时, 仍然可以互相影响。

```go
func appendSli(sli []int) {
  // 0xc000090000 0xc00000c060 [2]
	fmt.Printf("slice address in appendSli1:%p %p %+v\n", sli, &sli, sli) 
	sli[0] = 4
	sli = append(sli, 3)
  // 0xc000090030 0xc00000c060 [4 3]
  // sli发生了扩容->sli有了新的底层数组->%p sli的地址变了
  // 但此时&sli并没有变化, 因为sli变量还是不变的(slice是个struct), 变的是底层数组
	fmt.Printf("slice address in appendSli1:%p %p %+v\n", sli, &sli, sli)  
  // 这里再次修改sli的值, 已经不能影响到外部sli变量了
  // 因为此时本函数内部的sli变量对应的底层数组和main的sli变量对应的底层数组已经不是同一个数组了
	sli[0] = 5 
}
func main() {
	s := make([]int, 0, 1)
	s = append(s, 1)
	fmt.Printf("%p %p %+v\n", s, &s, s)  // 0xc000090000 0xc00008e000 [1]
	s[0] = 2
	appendSli(s)
	fmt.Printf("%p %p %+v\n", s, &s, s)  // 0xc000090000 0xc00008e000 [4]
}
```

我们可以通过函数修改存储元素的内容，但是永远修改不了`len`和`cap`，因为他们只是一个拷贝，如果要修改，那就要传递`*slice`作为参数才可以。

从slice创建slice的问题，[参考](https://www.cnblogs.com/qcrao-2018/p/10631989.html)

