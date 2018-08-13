# Golang依赖管理利器dep

## 定位
Dep是为开发人员开发的

> Dep is a tool intended primarily for use by **developers**, to support the work of actually writing and shipping code. It is not intended for end users who are installing Go software - that's what *go get* does.


## 安装与卸载

```
# Install Binary
$ curl https://raw.githubusercontent.com/golang/dep/master/install.sh | sh

# Install On MacOS
$ brew install dep
$ brew upgrade dep


# Uninstall Binary
$ rm $GOPATH/bin/dep

# Uninstall On MacOS
$ brew uninstall dep
```


## 创建新的项目

```
# 1. 创建项目目录，并进入
$ mkdir -p $GOPATH/src/github.com/alazyer/tutorial && cd $GOPATH/src/github.com/alazyer/tutorial

# 2. 初始化项目
`$ dep init`

# 3. 查看生成的目录结构
$ ls
Gopkg.toml Gopkg.lock vendor/
```

执行完上述三步后，其实已经有了一个使用dep管理依赖的Go项目了，这时可以添加版本控制支持，例如使用git初始仓库`git init`

如果需要添加依赖的话，可以执行下面这条命令

`$ dep ensure -add github.com/golang/mock`

执行完该命令后，会在项目下的`vendor`目录下保存所需要的依赖

```
# 查看
```

日常使用的话，dep除了上面提到的`dep ensure`之外，还有另外一个命令`dep status`


## Dep日常使用

dep提供了一些常用的命令，这些命令包括：

    1. dep init
    2. dep ensure
    2. dep status
    3. dep check 

### Dep ensure

通常使用`dep ensure`的有以下四个场景：

- **添加新的依赖**
如果在项目中引用了新的包，则可以直接使用`dep ensure`来添加依赖（官方说和goimports配合可能会有问题）
所以最好使用推荐的方式，`dep ensure -add xxx`

- **更新已有依赖**
`dep ensure --update [github.com/foo/bar]`


- **匹配项目中包引用发生变化**


- **匹配`Gopkg.tmol`中变化**

### Dep check

使用`dep check`命令用于查看，项目中是否有包引入变化、`Gopkg.toml`发生变化
