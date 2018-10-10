# viper学习笔记

## viper简介

[viper](https://github.com/spf13/viper)为Golang应用提供了一个完整的配置管理解决方案。能够处理所有的配置管理的需要和格式。以下是一些特性：

- 配置项不区分大小写
- 支持设置默认值
- 可以从json、toml、yaml、hcl和java配置文件读取配置项
- 可以从环境变量获取配置项
- 可以从其他配置系统（etcd或者consul）获取配置项
- 可以从命令行参数获取配置项值
- 实时监控，当配置文件发生变化时重新加载（可选）
- 支持参数别名（alias）



作为配置管理工具，肯定有两个功能需要了解：

1. 设置配置项
2. 读取配置项



viper支持多种方式设置配置项值，其优先级排序：

1. 显示的调用Set
2. flag
3. 环境变量
4. 配置文件
5. key/value store（etcd或者consul）
6. 默认值



## 设置配置项值

### 设置默认值

为配置项设置默认值，需要调用`viper.SetDefault`方法

```
viper.SetDefault("ContentDir", "contents")
viper.SetDefault("Taxonomies", map[string]string{"tag": "tags", "category": "categories"})
```



### 读取配置文件

使用配置文件获取配置项时，需要用到下面一些方法：

- 指定配置文件名称，`viper.SetConfigName`
- 添加配置文件寻找路径，`viper.AddConfigPath`
- 读取配置文件，`viper.ReadInConfig`

```
viper.SetConfigName("config") // name of config file (without extension)
viper.AddConfigPath("/etc/appname/")   // path to look for the config file in
viper.AddConfigPath("$HOME/.appname")  // call multiple times to add many search paths
viper.AddConfigPath(".")               // optionally look for config in the working directory
err := viper.ReadInConfig() // Find and read the config file
if err != nil { // Handle errors reading the config file
	panic(fmt.Errorf("Fatal error config file: %s \n", err))
}
```



### 从环境变量获取

通过环境变量获取配置项值时，**环境变量是区分大小写的，需要环境变量大写**，会用到下面这些方法：

+ 设置前缀，相当于命名空间作用，`viper.SetEnvPrefix`
+ 自动的从环境变量中获取值，`viper.AutomaticEnv`
+ 绑定配置项到环境变量，`viper.BindEnv`
+ 设置配置项名称替换规则，`viper.SetEnvKeyReplacer`

```
viper.SetEnvPrefix("alazyer")  // 设置环境变量时，需要大写

viper.BindEnv("name")
viper.BindEnv("age")

viper.AutomaticEnv()

os.Setenv("ALAZYER_NAME", "ylzhang")

viper.Get("alazyer_name")  // ylzhang

viper.SetEnvKeyReplacer(strings.NewReplacer("-", "_"))

viper.Get("alazyer-name")  // ylzhang
```



### 调用Set方法

`viper.Set`方法有最高的优先级

```
viper.RegisterAlias("loud", "Verbose")

viper.Set("verbose", true)
viper.Set("loud", true)
viper.GetBool("loud")
viper.GetBool("verbose")
```



### 通过Flag或者key/value仓库获取

TODO



## 获取配置项值

设置配置项的值的目的肯定是为了在其他地方使用，viper提供了若干方法用于获取配置项的值：

- `Get(key string): interface{}`
- `GetBool(key string) : bool`
- `GetFloat64(key string) : float64`
- `GetInt(key string) : int`
- `GetString(key string) : string`
- `GetStringMap(key string) : map[string]interface{}`
- `GetStringMapString(key string) : map[string]string`
- `GetStringSlice(key string) : []string`
- `GetTime(key string) : time.Time`
- `GetDuration(key string) : time.Duration`
- `IsSet(key string) : bool`
- `AllKeys(): []string`
- `AllSettings() : map[string]interface{}`



1. 通过名称可以猜测出来的是`Get`是一个基础的方法，其他`GetXXX`方法在内部调用`Get`方法，然后做了类型转换
2. `IsSet`方法用户判断是否设置了给定名称的配置项；
3. `AllKeys`方法用于获取所有配置项名称
4. `AllSettings`用于获取所有的配置项





## 一些其他方法

```
// 获取使用的配置文件
viper.ConfigFileUsed()

// 设置配置项别名
viper.RegisterAlias(alias, key)
```

