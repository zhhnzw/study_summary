## 测试

所有以`_test.go`为后缀名的源代码文件都是`go test`测试的一部分，不会被`go build`编译到最终的可执行文件中，`xxx_test.go`测试文件代码和被测试的源代码文件`xxx.go`放在同一个包内，一一对应。

### 单元测试

比如项目下有某个包内有个源代码文件`xxx.go`有这么一个函数：

```go
func lengthOfNonRepeatingSubStr(s string) int {
	lastOccurred := make(map[rune]int)
	start := 0
	maxLength := 0

	for i, ch := range []rune(s) {
		if lastI, ok := lastOccurred[ch]; ok && lastI >= start {
			start = lastI + 1
		}
		if i-start+1 > maxLength {
			maxLength = i - start + 1
		}
		lastOccurred[ch] = i
	}

	return maxLength
}
```

那么单元测试的文件`xxx_test.go`就放在相同的包内，测试函数的名字必须以`Test`开头，可选的后缀名必须以大写字母开头，需要一个`*testing.T`类型的参数，可以像下面这样按组测试，又称表格测试法：

```go
func TestSubstr(t *testing.T) {
	tests := []struct {
		s   string
		ans int
	}{
		// Normal cases
		{"abcabcbb", 3},
		{"pwwkew", 3},

		// Edge cases
		{"", 0},
		{"b", 1},
		{"bbbbbbbbb", 1},
		{"abcabcabcd", 4},

		// Chinese support
		{"一二三二一", 3},
		{"黑化肥挥发发灰会花飞灰化肥挥发发黑会飞花", 8},
	}

	for _, tt := range tests {
		actual := lengthOfNonRepeatingSubStr(tt.s)
		if actual != tt.ans {
			t.Errorf("got %d for input %s; "+
				"expected %d",
				actual, tt.s, tt.ans)
		}
	}
}
```

在项目根目录下直接执行`go test`就可以执行该项目下所有以`_test.go`为后缀的以`Test`开头的单元测试函数。

### 基准测试

基准测试就是在一定的工作负载之下检测程序性能的一种方法。基准测试函数以`Benchmark`为前缀，需要一个`*testing.B`类型的参数。

```go
func BenchmarkSubstr(b *testing.B) {
	s := "黑化肥挥发发灰会花飞灰化肥挥发发黑会飞花"
	for i := 0; i < 13; i++ {
		s = s + s
	}
	b.Logf("len(s) = %d", len(s))
	ans := 8
	b.ResetTimer() // 重置时间起始计算点，上面参数的初始化不计入

	for i := 0; i < b.N; i++ { // b.N 的值是系统根据实际情况去调整的
		actual := lengthOfNonRepeatingSubStr(s)
		if actual != ans {
			b.Errorf("got %d for input %s; "+
				"expected %d",
				actual, s, ans)
		}
	}
}
```

在项目根目录下直接执行`go test -bench=MethodName`就可以执行指定的基准测试函数。

可以为基准测试添加`-benchmem`参数，来获得内存分配的统计数据：

```bash
zhhnzw$ go test -v -bench=Substr -benchmem
=== RUN   TestSubstr
--- PASS: TestSubstr (0.00s)
goos: darwin
goarch: amd64
cpu: Intel(R) Core(TM) i7-7920HQ CPU @ 3.10GHz
BenchmarkSubstr
    nonrepeating_test.go:41: len(s) = 491520
    nonrepeating_test.go:41: len(s) = 491520
    nonrepeating_test.go:41: len(s) = 491520
BenchmarkSubstr-8            231           5152279 ns/op          655590 B/op          2 allocs/op
PASS
```

* `BenchmarkSubstr-8`中数字`8`表示`GOMAXPROCS`的值
* 数字`231`的含义：是本次`Benchmark`循环体共执行的次数
* `5152279 ns/op：`本次`Benchmark`循环体每循环一次耗费`5152279`纳秒
* `655590 B/op：`本次`Benchmark`循环体每循环一次分配了`655590`字节
* `2 allocs/op`：本次`Benchmark`循环体每循环一次进行了2次内存分配

### http服务测试

以下以`controller/v1`模块的单元测试为例，[完整项目](https://github.com/zhhnzw/demo)

```go
package v1

import (
	"demo/dao/mysql"
	"demo/settings"
	"github.com/gin-gonic/gin"
	"github.com/stretchr/testify/assert"
	"net/http"
	"net/http/httptest"
	"testing"
	"fmt"
)

// 需要用数据库的单元测试需要对其初始化
func init() {
	cfg := settings.MySQLConfig{
		Host:         "127.0.0.1",
		User:         "root",
		Password:     "root",
		DbName:       "test",
		Port:         3306,
		MaxOpenConns: 200,
		MaxIdleConns: 50,
	}
	if err := mysql.Init(&cfg); err != nil {
		fmt.Printf("init mysql failed, err:%v\n", err)
		return
	}
}

func TestGetUsers(t *testing.T) {
	gin.SetMode(gin.TestMode)
	r := gin.Default() // 如果直接引用 routers 模块，会造成循环引用，所以在此初始化一个 default
	url := "/v1/user"
	r.GET(url, GetUsers)
	req, _ := http.NewRequest(http.MethodGet, url, nil)
	w := httptest.NewRecorder()
	r.ServeHTTP(w, req)
	assert.Equal(t, 200, w.Code)
}
```

### 测试覆盖率

1. 取得`coverprofile`：`go test -v -coverprofile=coverage.out`
2. 查看：`go tool cover -html=coverage.out`
