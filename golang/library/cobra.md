## cobra

### 安装

Cobra:

```bash
$ go get -u github.com/spf13/cobra
```

Cobra代码生成工具:

```bash
$ go get -u github.com/spf13/cobra/cobra
```

### 项目自动生成

```bash
$ mkdir cobra-demo
$ cd cobra-demo/
$ cobra init --pkg-name cobra-demo
$ go mod init cobra-demo
$ go mod tidy
```

```bash
$ tree
.
├── LICENSE
├── cmd
│   └── root.go
├── go.mod
├── go.sum
└── main.go
```

### 命令行参数绑定

```go
// 生成的 cmd/root.go 中的 init() 有如下2行绑定 config 参数的代码
rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file (default is $HOME/.cobra-demo.yaml)")
rootCmd.Flags().BoolP("toggle", "t", false, "Help message for toggle")

// 添加类似于如下2行代码，可与 viper 来绑定 flags
rootCmd.PersistentFlags().StringVar(&apiVersion, "apiVersion", "", "apiVersion")
viper.BindPFlag("apiVersion", rootCmd.PersistentFlags().Lookup("apiVersion"))
```

在`cmd`目录创建对应的配置文件`sample.yaml`：

```yaml
apiVersion: zhhnzw.mock.com/v1
kind: CustomPod
```

和相应的`sample.go`：

```go
package cmd

type Config struct  {
	ApiVersion string
	Kind string
}

var Conf = new(Config)
```

在`cmd/root.go`添加解析到结构体变量的代码：

```go
// If a config file is found, read it in.
	if err := viper.ReadInConfig(); err == nil {
		fmt.Fprintln(os.Stderr, "Using config file:", viper.ConfigFileUsed())
		if err := viper.Unmarshal(Conf); err != nil {
			panic(fmt.Errorf("unmarshal conf failed, err:%s \n", err))
		}
		fmt.Println(Conf,rootCmd.PersistentFlags().Lookup("apiVersion").Value)
	}
```

打开`rootCMD`变量的`Run`函数，[项目代码](https://github.com/zhhnzw/k8s-demo/tree/main/cobra-demo)：

```go
Run: func(cmd *cobra.Command, args []string) {
		// 通过 cmd 变量从命令行读取参数，通过 viper 读取绑定的变量的参数
		fmt.Println("read from rootCmd Run:", cmd.Flag("config").Value, cmd.Flag("apiVersion").Value, viper.Get("apiVersion"))
	},
```

### 功能验证

```bash
# 可以看到cmd如期望的读取出了 config 参数
# 也如期望的绑定了 yaml 文件，读取出了 yaml 设置的默认的apiVersion 参数
$ go run main.go --config ./cmd/sample.yaml
Using config file: ./cmd/sample.yaml
bind Conf: &{zhhnzw.mock.com/v1 CustomPod}
read from rootCmd Run: ./cmd/sample.yaml  zhhnzw.mock.com/v1
# 通过 cmd 或者 viper 来读取，都成功取得了相同的参数值
$ go run main.go --config ./cmd/sample.yaml --apiVersion v2
Using config file: ./cmd/sample.yaml
bind Conf: &{v2 CustomPod}
read from rootCmd Run: ./cmd/sample.yaml v2 v2
```

### 添加子命令

```bash
$ cobra add sub
```

```go
// 可以看到，生成的 cmd/sub.go 已经将子命令添加到了 rootCmd 主命令
func init() {
	rootCmd.AddCommand(subCmd)
  // 可以为这个子命令添加专属的 Local Flag
}
```

修改 subCmd 变量：

```go
var subCmd = &cobra.Command{
	Use:   "sub",
	Short: "求和操作",
	Long: `实现输入参数的求和计算`,
	Run: func(cmd *cobra.Command, args []string) {
		sub := 0
		for k, e := range args {
			fmt.Println("第", k, "个变量：", e)
			v, err := strconv.Atoi(e)
			if err != nil {
				return
			}
			sub += v
		}
		fmt.Println("参数的求和为：", sub)
	},
}
```

### 功能验证

```bash
$ go run main.go sub 1 2 3
第 0 个变量： 1
第 1 个变量： 2
第 2 个变量： 3
参数的求和为： 6
```

### Command & Flag

如上 sub 为 Command，--config 和 --apiVersion 是 Flag

Command 可以按树型结构嵌套

### Persistent Flags & Local Flags

通过 PersistentFlags 绑定的参数在全局生效

Local Flags 绑定的参数在指定的 Command 下生效，在其他的 Command 下查询就查不出来了
