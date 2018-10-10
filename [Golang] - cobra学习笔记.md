# cobra学习笔记

## 一些概念

> **Commands** represent actions, **Args** are things and **Flags** are modifiers for those actions.
>
> Command is the central point of the application. Each interaction that the application supports will be contained in a Command. A command can have children commands and optionally run an action.
>
> A flag is a way to modify the behavior of a command. Cobra supports fully POSIX-compliant flags as well as the Go [flag package](https://golang.org/pkg/flag/). A Cobra command can define flags that persist through to children commands and flags that are only available to that command.



## 牛刀小试

通常Cobra-based应用都遵循下面的组织结构，可以通过`cobra`提供的命令行工具初始化一个项目`cd $GOPATH && cobra init github.com/alazyer/arboc`

```
$ tree github.com/alazyer/arboc
▾ arboc/
    ▾ cmd/
        root.go
      main.go
```

而`main.go`也很简单，只有一个目的：初始化Cobra，一般是下面这样子的

```go
package main

import (
  "github.com/alazyer/arboc/cmd"
)

func main() {
  cmd.Execute()
}
```

`cobra`还提供了另外一个有用的命令`cobra add`，执行该命令在后面加上自己想要的命令名字，然后就会在`cmd`下面看到以命令名字命名的文件了，只需要在上面进行一些定制就可以了，非常方便。以添加一个打印版本信息命令为例，执行`cobra add version`

```go
package cmd

import (
	"fmt"

	"github.com/spf13/cobra"
)

const VERSION = "v1.0"

// versionCmd represents the version command
var versionCmd = &cobra.Command{
	Use:   "version",
	Short: "Describe the version of arboc",
	Long: `Describe the version of this application`,
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println(rootCmd.Use + " " + VERSION)
	},
}

func init() {
	rootCmd.AddCommand(versionCmd)
}
```

到这里其实已经可以运行了

```shell
$ go run main.go
$ go run main.go version
```

