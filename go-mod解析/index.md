# go mod解析



## 关于go mod和内部包import

编写项目的时候遇到一个问题，即项目a在GOPATH/src中，a下有bc两个文件夹，b中的1.go需要调用c中的2.go。按照传统的import方法，应通过src下的相对路径进行引用。

```go
import "a/c"
```

但此时如果如此编写，goland会提示无法reslove目录，即找不到对应ac的位置。最先项目没有放在src中，考虑是否因为没有放在GOPATH中导致无法找到，遂将项目转移到src下，问题并未解决。

考虑是否goland配置有问题，查看全局gopath与项目gopath，并无问题，最终考虑go mod。

发现关闭go mod之后，能够正常import内部包，但由于项目基于gin框架，草率解决问题显然有些敷衍，所以继续研究go mod的具体效用，最终解决。（初学对go mod等一知半解，问题比较基础）

## 1、go mod

关于go mod的系统介绍和官方说明，请借鉴以下文章

1. [官方文档](https://github.com/golang/go/wiki/Modules) (github)
2. [Roberto Selbach](https://roberto.selbach.ca/intro-to-go-modules/)

Go mod即go module，是go 1.1.1版本发布的新模块特性，根本目的是为了移除对GOPATH和go get的依赖

简单来说，是方便把代码**放在src**之外～

 ### 介绍

module是**相关go包**的集合，是代码更替和版本控制的单元。模块主要标识为源文件夹下的go.mod文件，此文件目录也成为模块根，其取代了旧的基于src的方法，使用module的模式来指定源文件和导入包。

### go.mod文件解读

go.mod文件定义了**module**路径和其他需要在build时引入的模块的版本。

如下创建一个新项目**gin-test**，通过这个项目来说明go mod的具体细节。

![image-20200730223448694](https://tva1.sinaimg.cn/large/007S8ZIlgy1gh9fiunrv2j30h102640o.jpg)

首先新建文件夹**gin-test** 并在文件夹中使用go mod init命令进行模块初始化，注意此时指定了模块名为**gin-test-mod**。

打开**go.mod**文件查看内容：

![image-20200730223644933](https://tva1.sinaimg.cn/large/007S8ZIlgy1gh9fiy5t4oj30ck02c0u7.jpg)

**go.mod**第一行为模块名，可见模块名为**gin-test-mod**，之后第三行表示当前go的版本。

此时使用**go get**命令去远程拉取gin的相关包

![image-20200730224237324](https://tva1.sinaimg.cn/large/007S8ZIlgy1gh9fj3erovj30mp07i7ek.jpg)

可见，**go get**命令成功拉取到了**gin-gonic**相关的包，此时再查看**go.mod**

![image-20200730224412281](https://tva1.sinaimg.cn/large/007S8ZIlgy1gh9fj94d3kj30o70ax7i5.jpg)

可见此时go.mod中多了一些内容，包括一个require指令和内部一些相关依赖。

以最后一条举例

```mod
gopkg.in/yaml.v2 v2.3.0 //indirect
```

require命令中，声明module依赖gopkg.in/yaml.v2这个包，同时依赖特定版本，即2.3.0

其中indirect较为复杂，参看[go.mod 文件中的indirect准确含义](https://my.oschina.net/renhc/blog/3162751)

### go.mod内部命令

go.mod中除了上述的require命令还有exclude，replace命令，其分别表示排除某些包的特别版本，以及取代当前项目中的某些依赖包。

```go
require other/xxx 	v1.0.0
require new/xxx		v2.0.0
exclude old/xxx 		v1.0.0
replace bad/xxx 		v1.0.0 	=> good/thing v2.0.0
```

***

### go mod 命令

go mod提供了诸多命令来操作modules，同时如上述例子，在使用go get一些命令时，会自动后台对go.mod进行更新。

命令语法：**go mod &lt;command&gt; [arguments]**

* download : 下载模块到本地缓存，即GOCACHE
* edit : 编辑go.mod文件
* graph : 打印模块需求
* init : 初始化新的模块
* tidy : 添加缺失模块，移除多余模块
* verify : 验证依赖
* why : 解释包或模块 

***

### GO111MODULE 环境变量

**GO111MODULE** 是module支持的临时环境变量，其可以设置：off， on， auto。

设置为off时，将使用原始的GOPATH模式，即在gopath中查找依赖。

设置为on，则使用mod功能，不访问gopath。

设置为auto即默认模式，此时go会根据当前目录决定是否使用modules功能，若当前目录在src外切包含go.mod文件，才会启用mod功能。





## modules模式下的import引用

继续上面的例子，首先创建文件夹sum-test，并创建sum-func.go实现一个简单的sum函数

![image-20200730232243334](https://tva1.sinaimg.cn/large/007S8ZIlgy1gh9fjdfdjsj307j03smxa.jpg)

此时进入gin-test文件夹，创建main.go并尝试调用sum函数

![image-20200730232356820](https://tva1.sinaimg.cn/large/007S8ZIlgy1gh9fjgd2j9j30ck05mgm4.jpg)

此时能够正确运算结果，总结如下

* 使用modules的情况下，**不能按照原先gopath的方式引用内部包**，需要通过**[modules name]/[dir]**的方式来引用。如上述**module name为gin-test-mod**，引用sum-test文件夹下的sum-func.go中的Sum函数，需要通过**gin-test-mod/sum-test**的方式来引用。（**注意要使用modules名，使用gin-test目录名无效**）

  ![目录结构](https://tva1.sinaimg.cn/large/007S8ZIlgy1gh9fjjq7rhj306o03dwen.jpg)

* 当包名包含-符号时，如sum-test，在.go中，要转化为_下划线，即sum_test
* 若希望跨包调用函数，需将函数名首字母大写
* 欢迎点赞收藏加关注！！如有错误请指正，非常感谢！



