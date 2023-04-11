+++

author = "旅店老板"
title = "cobra从入门到实战"
date = "2023-03-02                                                                                                                                                                                                                                                                                                                                            "
description = "介绍cobra的基础知识和使用方式"
tags = [
	"redis",
]
categories = [
    "redis",
]
series = [""]
aliases = ["migrate-from-jekyl"]
image = "1.jpg"
mermaid = true
+++
## cobra是什么
cobra是一种创建强大的现代CLI应用程序的库。许多Go项目都应用了该库，如k8s、docker、istio、helm、hugo等，如果你要学习k8s源码或开发一个命令行工具，学习并使用cobra是一个不错的选择。

接下来通过一个example来说明cobra应该如何使用，该例子的需求为:开发一个sql导出命令行工具，用于导出多种数据库的转储文件
## 开始之前
安装`cobra-cli`命令行工具，命令如下：
```shell
go install github.com/spf13/cobra-cli@latest
```
`cobra-cli`是一个cobra的脚手架工具，用于快速搭建应用

## export导出工具开发
### 创建cobra项目
* 创建项目目录
```shell
mkdir export
```
***
* 进入项目目录
```shell
cd export
```
***
* 生成`go.mod`文件
```shell
go mod init export
```
也可直接通过ide创建Go module项目
***
* 初始化为cobra项目
```shell
cobra-cli init
```
该命令会在export目录下生成`main.go`文件和`cmd`目录，`cmd`目录下是`root.go`文件。  
`main.go`内容如下(main方法中调用了cmd中的Execute方法)：
```go
package main

import "export/cmd"

func main() {
	cmd.Execute()
}
```
`root.go`内容如下所示：
```go
package cmd

import (
	"github.com/spf13/cobra"
	"os"
)

var rootCmd = &cobra.Command{
	Use:   "export", 
	Short: "export是一款sql导出工具",
	Long: `export是一款由cobra开发的命令行工具，用于导出多种数据库的转储sql文件`,
}

func Execute() {
	err := rootCmd.Execute()
	if err != nil {
		os.Exit(1)
	}
}

func init() {
	rootCmd.Flags().BoolP("toggle", "t", false, "Help message for toggle")
}
```
1. root.go文件中创建了一个`cobra.Command`对象,该对象的`Use`字段为使用的命令，`Short`字段为简略的描述，`Long`字段为详细的描述(Short、Long的值已修改)
2. Execute函数在mao.go中被调用，它的作用是调用`cobra.Command`对象的Execute函数
3. init函数中为`cobra.Command`对象配置了一个参数(作用在后面说明)
***
### 添加查询版本的命令
通常命令行工具都可以通过命令打印自身的版本信息  
* 使用`cobra-cli add xxx`格式添加子命令：
```shell
cobra-cli add version
```
该命令会在`cmd`目录下下生成`version.go`文件,内容如下：
```go
package cmd

import (
	"fmt"
	"github.com/spf13/cobra"
)

var versionCmd = &cobra.Command{
	Use:   "version",
	Short: "打印版本信息",
	Long: `打印export自身的版本信息`,
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("export version 1.0.0")
	},
}

func init() {
	rootCmd.AddCommand(versionCmd)
}
```
1. `version.go`文件与`root.go`文件类似，也生成了一个`cobra.Command`对象，该对象多了一个`Run`字段，该字段的值为该命令具体的执行逻辑，已修改为打印`export version 1.0.0`
2. init函数中使用`rootCmd.AddCommand(versionCmd)`表示rootCmd添加命令versionCmd，即versionCmd为rootCmd的子命令，也就是version是export的子命令
***
* `go build`在windows环境下生成export.exe,执行`export.exe`命令展示内容如下：
```shell
$ ./export.exe
export是一款由cobra开发的命令行工具，用于导出多种数据库的转储sql文件

Usage:
  export [command]

Available Commands:
  completion  Generate the autocompletion script for the specified shell
  help        Help about any command
  version     打印版本信息

Flags:
  -h, --help     help for export
  -t, --toggle   Help message for toggle

Use "export [command] --help" for more information about a command.
```
1. 显示的结果中已经有了version命令，`-h`或`--help`和help是cobra应用自带的，用于查询帮助信息
2. `-t`或`--toggle`是在root.go的init()中添加的
* 执行`export.exe version`命令显示内容如下：
```shell
$ ./export.exe version
export version 1.0.0
```
该命令执行我们定义的Run函数，正确地打印了定义的版本信息
### 添加`mysql`和`oracle`子命令
* 执行cobra-cli命令生成对应文件
```shell
cobra-cli add mysql
```
```shell
cobra-cli add oracle
```
修改后`mysql.go`内容如下:
```go
package cmd

import (
	"fmt"
	"github.com/spf13/cobra"
)

var mysqlCmd = &cobra.Command{
	Use:   "mysql",
	Short: "该命令用于导出mysql的转储sql   你可使用export mysql -h 查看更多帮助 ",
	Long: `该命令用于导出mysql的转储sql   你可使用export mysql -h 查看更多帮助`,
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("mysql数据库连接成功")
		fmt.Println(fmt.Sprintf("ip:%s port:%s username:%s password:%s",ip,port,username,password))
		fmt.Println("sql文件导出完成")
	},
}

func init() {
	rootCmd.AddCommand(mysqlCmd)
}
```
修改后`oracle.go`内容如下:
```go
package cmd

import (
	"fmt"
	"github.com/spf13/cobra"
)

var oracleCmd = &cobra.Command{
	Use:   "oracle",
	Short: "该命令用于导出oracle的转储sql   你可使用export oracle -h 查看更多帮助 ",
	Long: `该命令用于导出oracle的转储sql   你可使用export oracle -h 查看更多帮助`,
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("oracle数据库连接成功")
		fmt.Println(fmt.Sprintf("ip:%s port:%s username:%s password:%s",ip,port,username,password))
		fmt.Println("sql文件导出完成")
	},
}

func init() {
	rootCmd.AddCommand(oracleCmd)
}
```
Run函数中没有真实地导出逻辑，我们用打印代替，打印了ip、port、username、password信息，参数的定义会在下面详细说明
* 绑定参数  
  数据库连接信息(ip、port、username、password、database)，需要通过命令行工具传入，我们在`root.go`中定义我们需要的参数，这些参数无论是mysql还是oracle数据库都需要,
  因此我们将这些参数绑定到`rootCmd`对象中

最终`root.go`内容如下：
```go
package cmd

import (
	"github.com/spf13/cobra"
	"os"
)

var ip string
var port string
var username string
var password string
var database string
var out_dir string

var rootCmd = &cobra.Command{
	Use:   "export",
	Short: "export是一款sql导出工具",
	Long: `export是一款由cobra开发的命令行工具，用于导出多种数据库的转储sql文件`,
}

func Execute() {
	err := rootCmd.Execute()
	if err != nil {
		os.Exit(1)
	}
}

func init() {
	rootCmd.PersistentFlags().StringVarP(&ip,"ip","i" , "localhost", "主机名")
	//&ip:表示传入的参数具体绑定的对象
	//ip：表示传入的参数名称
	//i： 表示传入的参数名称的简称
	//localhost：表示为不传ip参数的默认值为localhost
	//主机名：表示参数中文名
	rootCmd.PersistentFlags().StringVar(&port, "port", "3306", "端口")
	rootCmd.PersistentFlags().StringVarP(&username, "username","u" , "root", "用户名")
	rootCmd.PersistentFlags().StringVarP(&password, "password", "p" ,"root", "密码")
	rootCmd.PersistentFlags().StringVar(&database, "database","first", "数据库名")
	rootCmd.PersistentFlags().StringVarP(&out_dir, "out_dir","o" , "./data/", "文件输出目录")
}
```
***
* `go build`在windows环境下生成export.exe,测试该工具展示内容如下：
```shell
$ ./export.exe mysql -h
该命令用于导出mysql的转储sql   你可使用export mysql -h 查看更多帮助

Usage:
  export mysql [flags]

Flags:
  -h, --help   help for mysql

Global Flags:
      --database string   数据库名 (default "first")
  -i, --ip string         主机名 (default "localhost")
  -o, --out_dir string    文件输出目录 (default "./data/")
  -p, --password string   密码 (default "root")
      --port string       端口 (default "3306")
  -u, --username string   用户名 (default "root")
```
绑定到`rootCmd`的都显示为`Global Flags:`
```shell
$ ./export.exe mysql -i 192.168.0.1
mysql数据库连接成功
ip:192.168.0.1 port:3306 username:root password:root
sql文件导出完成
```
我们只传入了`-i`的参数，其他都为默认值
***
### 添加`table`参数
假设现在需求变更，在导出mysql的转储文件时，如果有table参数就导出指定表的数据，而不用导出整个数据库
* `root.go`添加`table`参数,内容如下：
```go
package cmd

import (
	"github.com/spf13/cobra"
	"os"
)

var ip string
var port string
var username string
var password string
var database string
var out_dir string
var table string

var rootCmd = &cobra.Command{
	Use:   "export",
	Short: "export是一款sql导出工具",
	Long: `export是一款由cobra开发的命令行工具，用于导出多种数据库的转储sql文件`,
}

func Execute() {
	err := rootCmd.Execute()
	if err != nil {
		os.Exit(1)
	}
}

func init() {
	rootCmd.PersistentFlags().StringVarP(&ip,"ip","i" , "localhost", "主机名")
	rootCmd.PersistentFlags().StringVar(&port, "port", "3306", "端口")
	rootCmd.PersistentFlags().StringVarP(&username, "username","u" , "root", "用户名")
	rootCmd.PersistentFlags().StringVarP(&password, "password", "p" ,"root", "密码")
	rootCmd.PersistentFlags().StringVar(&database, "database","first", "数据库名")
	rootCmd.PersistentFlags().StringVarP(&out_dir, "out_dir","o" , "./data/", "文件输出目录")
}
```
***
* 绑定`table`参数
因为table参数的flag是mysql命令独有的，因此把该参数绑定到`mysqlCmd`对象上
修改后`mysql.go`内容如下：
```go
package cmd

import (
	"fmt"
	"github.com/spf13/cobra"
)

var mysqlCmd = &cobra.Command{
	Use:   "mysql",
	Short: "该命令用于导出mysql的转储sql   你可使用export mysql -h 查看更多帮助 ",
	Long: `该命令用于导出mysql的转储sql   你可使用export mysql -h 查看更多帮助`,
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("mysql数据库连接成功")
		fmt.Println(fmt.Sprintf("ip:%s port:%s username:%s password:%s",ip,port,username,password))
		fmt.Println("sql文件导出完成")
	},
}

func init() {
	mysqlCmd.PersistentFlags().StringVarP(&table, "table", "t" ,"", "表名(如不指定表名,则导出该库下全部)")
	rootCmd.AddCommand(mysqlCmd)
}
```
***
* `go build`在windows环境下生成export.exe,测试该工具展示内容如下：
```shell
$ ./export.exe mysql -h
该命令用于导出mysql的转储sql   你可使用export mysql -h 查看更多帮助

Usage:
  export mysql [flags]

Flags:
  -h, --help           help for mysql
  -t, --table string   表名(如不指定表名,则导出该库下全部)

Global Flags:
      --database string   数据库名 (default "first")
  -i, --ip string         主机名 (default "localhost")
  -o, --out_dir string    文件输出目录 (default "./data/")
  -p, --password string   密码 (default "root")
      --port string       端口 (default "3306")
  -u, --username string   用户名 (default "root")
```
`Flags`下增加了`-t`的flag
```shell
$ ./export.exe oracle -h
该命令用于导出oracle的转储sql   你可使用export oracle -h 查看更多帮助

Usage:
  export oracle [flags]

Flags:
  -h, --help   help for oracle

Global Flags:
      --database string   数据库名 (default "first")
  -i, --ip string         主机名 (default "localhost")
  -o, --out_dir string    文件输出目录 (default "./data/")
  -p, --password string   密码 (default "root")
      --port string       端口 (default "3306")
  -u, --username string   用户名 (default "root")
```
`table`参数对`oracle`命令是不可见的,实现了对应的需求

export项目Demo源码地址：[https://github.com/ldlb9527/export](https://github.com/ldlb9527/export 'export')



