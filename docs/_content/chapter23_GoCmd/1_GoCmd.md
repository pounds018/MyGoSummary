## 1. go相关命令总结:

### 1.1 系统命令

1. `go version`: 查看go版本       

2. `go env`: 查看go环境变量

    - -w 变量名:  设置指定环境变量

3. `go build`: 编译

    - 在go文件相同目录下执行命令: `go build `
    - 在go文件不同目录下执行命令: `go build 文件路径(src中第一级文件夹开始)`
    - 编译的时候设置exe文件名字: `go build -o 想要设置的名字.exe`(windows这样写,其他可以不加exe)
    - 编译的时候查看时候有资源竞争存在: `go build -race 源码文件名字`
    > go build命令在哪儿执行生成的exe文件就在哪儿

4. `go run`: 像执行脚本文件一样执行go代码

5. `go install`: 编译并运行,可以在任何地方执行
    - 先编译得到exe文件
    - 将可执行文件拷贝到 `GOPATH/bin`下面

6. `gofmt` : 格式化.go文件代码格式

### 1.2 依赖管理:  

1. `godep`:

   ```BASH
   godep save     将依赖项输出并复制到Godeps.json文件中
   godep go       使用保存的依赖项运行go工具
   godep get      下载并安装具有指定依赖项的包
   godep path     打印依赖的GOPATH路径
   godep restore  在GOPATH中拉取依赖的版本
   godep update   更新选定的包或go版本
   godep diff     显示当前和以前保存的依赖项集之间的差异
   godep version  查看版本信息
   ```

2. `go mod`

   ```bash
   go mod download    下载依赖的module到本地cache（默认为$GOPATH/pkg/mod目录）
   go mod edit        编辑go.mod文件
   go mod graph       打印模块依赖图
   go mod init        初始化当前文件夹, 创建go.mod文件
   go mod tidy        增加缺少的module，删除无用的module
   go mod vendor      将依赖复制到vendor下
   go mod verify      校验依赖
   go mod why         解释为什么需要依赖
   ```


### 1.3 测试命令:

`go test`: go语言单元测试的命令,他会运行当前目录下面的所有带 `_test`的go文件

`go test -v`: 查看测试函数名称和运行时间

`go test -run="正则表达式"`: 通过`run`参数中的正则表达式,来匹配要执行的测试函数

`go test -cover -coverprofile`:  查看覆盖率测试结果,`-coverprofile`参数可以将测试覆盖率结果输出到指定文件

`go tool cover -html=c.out` : 查看哪些测试代码被执行了

`go test -bench=函数名称` -benchmem: `-bench=函数名称`指定某个函数进行基准测试 ,`-benchmem`内存分配统计

 

