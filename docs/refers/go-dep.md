主要以[这篇文章](https://item.congci.com/-/content/golang-dep-baoguanli-tools-new)为主体进行了一点补充。

Go语言程序组织和构建的基本单元是Package，但Go语言官方却没有提供一款“像样的”Package Management Tool(包管理工具)。随着Go语言在全球范围内应用的愈加广泛，缺少官方包管理工具这一问题变得日益突出。

2016年GopherCon大会后，在Go官方的组织下，一个旨在改善Go包管理的commitee成立了，共同应对Go在package management上遇到的各种问题。经过各种脑洞和讨论后，该commitee在若干月后发布了“Package Management Proposal”，并启动了最有可能被接纳为官方包管理工具的项目dep的设计和开发。2017年年初，dep项目正式对外开放。截至目前，dep发布了v0.5.0版本，从[这里](https://github.com/golang/dep)以及 0.5.0的[releasenote](https://golang.github.io/dep/blog/)获取最新版内容。

## Go包管理的演进历史

### go get
go get 可以很方便的获取大量的再github上的package，但随着对Go语言使用的深入，发现go get给我们带来方便的同时，也带来了不少的麻烦。go get本质上是git、hg等这些vcs工具的高级wrapper。对于使用git的go package来说，go get的实质就是将package git clone到本地的特定目录下（$GOPATH/src），同时go get可以自动解析包的依赖，并自动下载相关依赖包。
go get机制的设计很大程度上源于Google公司内部的单一root的代码仓库的开发模式，并且似乎google内部各个project/repository的master分支上的代码都是被认为stable的，因此go get仅仅支持获取master branch上的latest代码，没有指定version、branch或revision的能力。而在Google公司以外的世界里，这样的做法会给gopher带来不便：依赖的第三方包总是在变。一旦第三方包提交了无法正常build或接口不兼容的代码，依赖方立即就会受到影响。

而gopher们又恰恰希望自己项目所依赖的第三方包能受到自己的控制，而不是随意变化。这样，godep、gb、glide等一批第三方包管理工具出现了。

以应用最为广泛的godep为例。为了能让第三方依赖包“稳定下来”，实现项目的reproduceble build，godep将项目当前依赖包的版本信息记录在Godeps/Godeps.json中，并将依赖包的相关版本存放在Godeps/_workspace中。在编译时(godep go build)godep通过临时修改GOPATH环境变量的方法让go编译器使用缓存在Godeps/_workspace下的项目依赖的特定版本的第三方包，这样保证了项目不再受制于依赖的第三方包的master branch上的latest代码的变动了。

不过，godep的“版本管理”本质上是通过缓存第三方库的某个revision的快照实现的，这种方式依然让人感觉难于管理。同时，通过对GOPATH的“偷梁换柱”的方式实现使用Godeps/_workspace中的第三方库的快照进行编译也无法兼容Go原生编译器，必须使用godep go xxx来进行。

为此，Go进一步引入vendor机制来减少gopher在包管理问题上的心智负担。

### verdor 机制
Go team也一直在关注Go语言包依赖的问题，尤其是在Go 1.5实现自举的情况下，官方同样在1.5版本中推出了vendor机制。vendor机制是Russ Cox在Go 1.5发布前期以一个experiment feature身份紧急加入到go中的(go 1.6脱离experiment身份)。vendor标准化了项目依赖的第三方库的存放位置（不再需要Godeps/_workspace了），同时也无需对GOPATH环境变量进行“偷梁换柱”了，go compiler原生优先感知和使用vendor下缓存的第三方包。
- vendor是一个特殊的目录，在应用的源码目录下，go doc工具会忽略它。
- vendor机制支持嵌套vendor，vendor中的第三方包中也可以包含vendor目录。
- 若不同层次的vendor下存在相同的package，编译查找路径优先搜索当前pakcage下的vendor是否存在，若没有再向parent pacakge下的vendor搜索（x/y/z作为parentpath输入，搜索路径：x/y/z/vendor/path->x/y/vendor/path->x/vendor/path->vendor/path)
- 在使用时不用理会vendor这个路径的存在，该怎么import包就怎么import，不要出现import “d/vendor/p”的情况。vendor是由go tool隐式处理的。
- 不会校验vendor中package的import path是否与canonical import路径是否一致了。

优缺点：
- 优点：可能解决不同的版本依赖冲突问题，不同的层次的vendor存放在不同的依赖包。
- 缺点：由于go的package是以路径组织的，在编译时，不同层次的vendor中相同的包会编译两次，链接两份，程序文件变大，运行期是执行不同的代码逻辑。会导致一些问题，如果在package init中全局初始化，可能重复初化出问题，也可能初化为不同的变量（内存中不同），无法共享获取。

不过即便有了vendor的支持，vendor内第三方依赖包的代码的管理依旧是不规范的，要么是手动的，要么是借助godep这样的第三方包管理工具。目前自举后的Go代码本身也引入了vendor，不过go项目自身对vendor中代码的管理方式也是手动更新，Go自身并未使用任何第三方的包管理工具。

从Go官方角度出发，官方go包依赖的解决方案的下一步就应该是解决对vendor下的第三方包如何进行管理的问题：依赖包的分析、记录和获取等，进而实现项目的reproducible build。dep就是用来做这事儿的。

## Dep 使用
### Dep 安装
**二进制安装**

> $ curl https://raw.githubusercontent.com/golang/dep/master/install.sh | sh

**源码安装**
```
go get -d -u github.com/golang/dep
cd $(go env GOPATH)/src/github.com/golang/dep
DEP_LATEST=$(git describe --abbrev=0 --tags)
git checkout $DEP_LATEST
go install -ldflags="-X main.version=$DEP_LATEST" ./cmd/dep
git checkout master
```

安装好了之后可以help一下，看看dep的基本使用：
```bash
root@ubuntu:~# dep help
Dep is a tool for managing dependencies for Go projects

Usage: "dep [command]"

Commands:

  init     Set up a new Go project, or migrate an existing one
  status   Report the status of the project's dependencies
  ensure   Ensure a dependency is safely vendored in the project
  version  Show the dep version information
  check    Check if imports, Gopkg.toml, and Gopkg.lock are in sync

Examples:
  dep init                               set up a new project
  dep ensure                             install the project's dependencies
  dep ensure -update                     update the locked versions of all dependencies
  dep ensure -add github.com/pkg/errors  add a dependency to the project

Use "dep help [command]" for more information about a command.



root@ubuntu:~# dep version
dep:
 version     : v0.5.0
 build date  : 2018-07-26
 git hash    : 224a564
 go version  : go1.10.3
 go compiler : gc
 platform    : linux/amd64
 features    : ImportDuringSolve=false
```
### Dep 卸载
如果是二进制安装的，直接`rm $GPPATH/bin/dep`即可

## Dep的基本工作流
### Dep init
如果想将一个包使用dep管理起来，直接在这个包中执行`dep init -v`，注意的是需要将你的放置在$GPPATH/src目录下。
执行完`dep init`之后，当前目录下会多出两个文件+一个目录：
```bash
root@ubuntu:/home/gang/go/src/test# ls
Gopkg.lock  Gopkg.toml  main.go  vendor
```
另外，如果想要对包进行版本控制，可以同时进行git的初始化。
dep init 大概进行了如下的操作：
- 利用gps分析当前代码包中的包依赖关系；
- 将分析出的项目包的直接依赖(即main.go显式import的第三方包，direct dependency)约束(constraint)写入项目根目录下的Gopkg.toml文件中；
- 将项目依赖的所有第三方包（包括直接依赖和传递依赖transitive dependency）在满足Gopkg.toml中约束范围内的最新version/branch/revision信息写入Gopkg.lock文件中；
- 创建root vendor目录，并且以Gopkg.lock为输入，将其中的包（精确checkout 到revision）下载到项目root vendor下面。

**Gppkg.toml**
存放通过gps分析出来以来的包及对应的版本约束，类似如下格式：
```bash
[[constraint]]
  branch = "master"
  name = "github.com/beego/mux" 

[prune]
  go-tests = true
  unused-packages = true
```

**Gopkg.lock**
生成的Gppkg.lock则记录了在toml中的约束下的所有依赖的可用的最新版本，形如：
```bash
# This file is autogenerated, do not edit; changes may be undone by the next 'dep ensure'.


[[projects]]
  branch = "master"
  digest = "1:5b239404e804d87bc853b21e8fd3c46bfddee62232c36eaad97b38a31f43ff8d"
  name = "github.com/beego/mux"
  packages = ["."]
  pruneopts = "UT"
  revision = "6660b4b5accbb383fac89498e57d3250d5e907ac"

[solve-meta]
  analyzer-name = "dep"
  analyzer-version = 1
  input-imports = ["github.com/beego/mux"]
  solver-name = "gps-cdcl"
  solver-version = 1
```

**vendor**
此时 vendor已经有了所有的依赖包，形如：
```
root@ubuntu:/home/gang/go/src/test# tree -L 2 vendor/
vendor/
└── github.com
    └── beego

2 directories, 0 files
```
dep init 完毕之后，相关的依赖包都已经被vendor， 可以使用go build/install来进行程序构建了。

如果对dep init 自动生成的Gopkg文件没有异议，就可以把该文件作为源代码的一部分提到代码库中。其他人下载了代码包之后，可以直接通过dep命令来下载依赖使用。对于verdor一般不作为源码包的一部分。

### dep ensure
dep ensure命令可以根据Gopkg.toml和Gopkg.lock中的数据构建vendor目录和同步里面的包。
```
root@ubuntu:/home/gang/go/src/test# dep ensure -v
# Bringing vendor into sync
(1/1) Wrote github.com/beego/mux@master: missing from vendor
```
通过dep status查看当前的以来情况。
```
root@ubuntu:/home/gang/go/src/test# dep status
PROJECT               CONSTRAINT     VERSION        REVISION  LATEST   PKGS USED
github.com/beego/mux  branch master  branch master  6660b4b   6660b4b  1  
```

如果手动修改了Gopkg.toml文件，那么需要运行`dep ensure --update`去同步Gopkg.lock和vendor下的数据。

还可以直接指定版本来安装，使用命令如下：
``` dep ensure 'github.com/xxx/xxx@1.1.1'```
