## 扩展k8s集群
k8s 高度的可配置，可扩展。
定制方法大概可以被分为两类：配置和扩展。配置包括修改启动项,本地配置文件或者API资源。扩展是通过运行额外的进程或者服务。
配置指的是在启动kube-apiserver、kube-controller-manager、kube-scheduler、kubelet等服务进程中可以通过指定不同的启动项达到不同的目的。一般来说，一个集群的启动项以及配置文件是比较少的更改的，
扩展集群指的是那些扩展的、和k8s本身深度集成的软件组件。用来支持新的type以及硬件种类。
### 扩展模式
k8s被设计为通过客户端编程来实现自动化。一种编写客户端程序的指定模式称为Controller模式。Controller通常读对象的.spec字段，做一些操作，然后更新对象的.status字段。
### 可扩展得点
1. kubelet侧，Kubectl plugins扩展了kubectl二进制，但是其只会影响到本地环境的用户，而不会对server端造成影响。
2. 一些apiserver侧的扩展点可以做一些鉴权的工作。
3. apiserver 服务多种资源，既有像pod这样的内置资源，还可以用户自定义资源。
4. scheduler是负责调度pod的，在scheduler中还有一些可以扩展的点。
5. 以不同的Controller来处理不同的自定义资源
6. 网络插件来实现对不同网络的对接。
7. 存储插件实现对不同存储的对接。

### API扩展
1. 自定义资源，以及Controller，以及对应的API
2. 当添加了新的API意味着添加了控制回路去读写这些API，将这些控制回路组合起来，新的API，新的资源对象，称之为一个operator。
3. 新的API是伴随着一个新的API group呈现的，所以一般不会影响到已有的API（比如操作pod的API），但是API ACCESS 扩展可以。


### 基础设备扩展
1. 存储插件
2. 设备插件
3. 网络插件
4. scheduler 扩展





## 扩展k8s API
### 用户自定义资源——Custom Resources
Resource在k8s API中是一个endpoint，存储了某种特定类型的API 对象的集合。比如，pods resources 包含了pod对象的集合。Custom Resource作为一个k8sAPI的扩展，是不需要对每个k8s集群都可用的。一旦一个CR 安装了，用户可以通过kubectl创建和访问他的对象

#### 增加CR
两种方式：
1. CRD
2. API Aggretation

#### CRD
使用CRD 可以创建一个新的CR
具体参见 CRD的使用一文

#### API server aggregation

#### 安装一个CR的准备

#### 访问一个CR



### api中的aggregation layer层



## 计算、存储、网络扩展




## Service Catalog
