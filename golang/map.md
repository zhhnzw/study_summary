## 哈希

Go 语言使用拉链法来解决哈希碰撞的问题

```go
type hmap struct {
	count     int
	flags     uint8
	B         uint8
	noverflow uint16
	hash0     uint32

	buckets    unsafe.Pointer
	oldbuckets unsafe.Pointer
	nevacuate  uintptr

	extra *mapextra
}
```

无论 `make` 是从哪里来的，只要我们使用 `make` 创建哈希，Go 语言编译器都会在类型检查期间将它们转换成对`runtime.makemap` 的调用，使用字面量来初始化哈希也只是语言提供的辅助工具，最后调用的都是 `runtime.makemap`

其他细节：

* 哈希在每一个桶(链表)中存储键对应哈希的前 8 位，当对哈希进行操作时，这些 `tophash` 就成为了一级缓存帮助哈希快速遍历桶中元素
* 每一个桶都只能存储 8 个键值对，一旦当前哈希的某个桶超出 8 个，新的键值对就会被存储到哈希的溢出桶中
* 随着键值对数量的增加，溢出桶的数量和哈希的装载因子也会逐渐升高，超过一定范围就会触发扩容
* 扩容会将桶的数量翻倍，元素再分配的过程也是在调用写操作时增量进行的，不会造成性能的瞬时巨大抖动

map的Go语言实现细节优化点很复杂, [参考](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/)

线程安全问题

