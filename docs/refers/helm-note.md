Helm是一个管理K8s charts的工具。而charts是一个预先配置好的k8s 资源包
Helm 就像一个应用商店，可以下载别人已经做好的charts 跑在自己的k8s上，也可以上传自己制作的charts来给别人使用。通过helm还可以对资源包进行版本管理和发布。可理解为helm就是k8s的yum/apit-get

包含的几个概念：
- chart: 一个 Helm 包，其中包含了运行一个应用所需要的镜像、依赖和资源定义等，还可能包含 Kubernetes 集群中的服务定义，类似 Homebrew 中的 formula，APT 的 dpkg 或者 Yum 的 rpm 文件
- release: 在 Kubernetes 集群上运行的 Chart 的一个实例。在同一个集群上，一个 Chart 可以安装很多次。每次安装都会创建一个新的 release。例如一个 MySQL Chart，如果想在服务器上运行两个数据库，就可以把这个 Chart 安装两次。每次安装都会生成自己的 Release，会有自己的 Release 名称。
- repository: 用于发布和存储 Chart 的仓库
通过上面几个概念，总结一下helm就是：helm在k8s上安装charts，每一次安装都是创建一个新的release，通过search repostiories来找到你想要的charts，你自己也可以发布一个charts


- Helm分为两部分：client端（helm）和server端（tiller）
- Tiller跑在k8s集群内部，负责管理charts
- Helm跑在本地，或者CI/CD，以及任何你需要的地方
- Charts是Helm管理resource package的形式，它至少包含两个东西：
    - 一个chart的描述文件：chart.yaml
    - 一个或多个模板，包含k8s的清单文件。
- charts 可以存在本地，也可以从远端chart 库中获取


### Helm的安装
从github获取最新release 软件包。可选的有源码包和二进制包
[这里](https://storage.googleapis.com/kubernetes-helm/helm-v2.10.0-linux-amd64.tar.gz)，然后解压即可得到二进制包。
执行`mv linux-amd64/heml /usr/local/bin/helm`

### 初始化Helm以及安装Tiller
在缺省配置下， Helm 会利用 "gcr.io/kubernetes-helm/tiller" 镜像在Kubernetes集群上安装配置 Tiller；并且利用 "https://kubernetes-charts.storage.googleapis.com" 作为缺省的 stable repository 的地址。由于在国内可能无法访问 "gcr.io", "storage.googleapis.com" 等域名,使用阿里云提供的镜像。
```bash
[root@rdb24 ~]# helm init --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.10.0 --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts --service-account tiller

```

### Helm基本使用

#### helm search
查询存储库可用的所有helm charts,默认 chart库名为stable
``` bash
[root@rdb24 ~]# helm search mysql 
[root@rdb24 ~]# helm search mysql
NAME                         	CHART VERSION	APP VERSION	DESCRIPTION                                                 
presslabs/mysql-operator     	0.1.12       	0.1.12     	A Helm chart for mysql operator                             
stable/mysql                 	0.3.5        	           	Fast, reliable, scalable, and easy to use open-source rel...
presslabs/orchestrator       	0.1.3        	3.0.11     	A Helm chart for github's mysql orchestrator                
stable/percona               	0.3.0        	           	free, fully compatible, enhanced, open source drop-in rep...
stable/percona-xtradb-cluster	0.0.2        	5.7.19     	free, fully compatible, enhanced, open source drop-in rep...
stable/gcloud-sqlproxy       	0.2.3        	           	Google Cloud SQL Proxy                                      
stable/mariadb               	2.1.6        	10.1.31    	Fast, reliable, scalable, and easy to use open-source rel...
...
```

#### helm inspect <chart>查看详情
```
[root@rdb24 ~]# helm inspect stable/mysql
description: Fast, reliable, scalable, and easy to use open-source relational database
  system.
engine: gotpl
home: https://www.mysql.com/
icon: https://www.mysql.com/common/logos/logo-mysql-170x115.png
keywords:
- mysql
- database
- sql
maintainers:
- email: viglesias@google.com
  name: Vic Iglesias
name: mysql
sources:
- https://github.com/kubernetes/charts
- https://github.com/docker-library/mysql
version: 0.3.5

---
## mysql image version
## ref: https://hub.docker.com/r/library/mysql/tags/
##
image: "mysql"
imageTag: "5.7.14"
...
```


#### repo 相关操作：

```bash
>helm repo update
>helm repo add stable https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
>helm repo list
>helm repo remove stable
>helm repo index
```

#### 删除tiller
```
kubectl delete deployment tiller-deploy --namespace kube-system
或者：helm reset
```

#### 安装/删除chart
```
[root@rdb24 ~]# helm install stable/mysql -n mysql-operator
[root@rdb24 ~]# helm ls
NAME          	REVISION	UPDATED                 	STATUS  	CHART                	APP VERSION	NAMESPACE     
mysql-operator	1       	Thu Aug 23 15:55:31 2018	DEPLOYED	mysql-0.3.5          	           	default       
presslab-mysql	1       	Thu Aug 23 09:54:54 2018	DEPLOYED	mysql-operator-0.1.12	0.1.12     	mysql-operator
[root@rdb24 ~]# helm delete mysql-operator
release "mysql-operator" deleted
[root@rdb24 ~]# helm ls
NAME          	REVISION	UPDATED                 	STATUS  	CHART                	APP VERSION	NAMESPACE     
presslab-mysql	1       	Thu Aug 23 09:54:54 2018	DEPLOYED	mysql-operator-0.1.12	0.1.12     	mysql-operator

```
#### 查看某个release的状态：
```bash
[root@rdb24 ~]# helm status mysql-operator1
LAST DEPLOYED: Thu Aug 23 16:15:27 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Secret
NAME                   TYPE    DATA  AGE
mysql-operator1-mysql  Opaque  2     20m

==> v1/PersistentVolumeClaim
NAME                   STATUS   VOLUME  CAPACITY  ACCESS MODES  STORAGECLASS  AGE
mysql-operator1-mysql  Pending  20m

==> v1/Service
NAME                   TYPE       CLUSTER-IP      EXTERNAL-IP  PORT(S)   AGE
mysql-operator1-mysql  ClusterIP  10.200.221.164  <none>       3306/TCP  20m

==> v1beta1/Deployment
NAME                   DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
mysql-operator1-mysql  1        1        1           0          20m
...
```
#### 定制化安装
可以在install的时候给release指定参数，可接收的参数通过`helm inspect values <charts>`来查看。然后编写yaml文件。
```
$ echo '{mariadbUser: user0, mariadbDatabase: user0db}' > config.yaml
$ helm install -f config.yaml stable/mariadb
或者使用--set
helm install --set mariadbUser=user0,xxxx=xxxx stable/mariadb
```
#### 更新及回退release
```
$ helm upgrade 
$helm rolllback
```

### helm chart
快速浏览：
``` 
helm create xxx 创建
helm package  打包package
helm insall xxx.tgz 安装打包好的charts

```

#### chart的组织结构
所有文件放到一个目录里，目录名就是chart名
```
wordpress/
  Chart.yaml          # 包含这个chart的描述信息
  LICENSE             # OPTIONAL: 该chart的license
  README.md           # OPTIONAL: A human-readable README file
  requirements.yaml   # OPTIONAL: 描述这个chart依赖的yaml file
  values.yaml         # 该chart默认的configuration value
  charts/             # 一个目录，包含任何该chart所依赖的charts
  templates/          # 一个目录，包含的是模板文件，和values结合起来可以生成k8s manifest file
  templates/NOTES.txt # OPTIONAL: A plain text file containing short usage notes
```

#### Chart.yaml
```
apiVersion: The chart API version, always "v1" (required)
name: The name of the chart (required)
version: A SemVer 2 version (required)，必须符合SemVer2 标准 (参考https://semver.org/lang/zh-CN/)
kubeVersion: A SemVer range of compatible Kubernetes versions (optional)
description: A single-sentence description of this project (optional)
keywords:
  - A list of keywords about this project (optional)(当helm search时就是搜索的keyword)
home: The URL of this project's home page (optional)
sources:
  - A list of URLs to source code for this project (optional)
maintainers: # (optional)
  - name: The maintainer's name (required for each maintainer)
    email: The maintainer's email (optional for each maintainer)
    url: A URL for the maintainer (optional for each maintainer)
engine: gotpl # The name of the template engine (optional, defaults to gotpl)
icon: A URL to an SVG or PNG image to be used as an icon (optional).
appVersion: The version of the app that this contains (optional). This needn't be SemVer.
deprecated: Whether this chart is deprecated (optional, boolean)
tillerVersion: The version of Tiller that this chart requires. This should be expressed as a SemVer range: ">2.0.0" (optional)
```
- version：k8s helm组织package名是按照名字加版本号的，比如name=nigix, version=1.2.3,那么最后的包名就是negix-1.2.3.tgz, `version`这个字段会被很多helm 工具使用到，要很好的规划
- apiVersion: 和version无关的字段，可选字段，无特定格式，起到一定说明内容。
- deprecated：表明这个chart是否被废除，废除一个chart的流程是：1. 更新chart.yaml文件，将deprecated置为true；2. 在chart 库发布新版本chart；3. 从源库删除该chart（例如git等）。
- license，
- 

#### Chart中的依赖（requirements.yaml和chart）
一个chart可能会依赖于多个chart，这些依赖可以动态的linke到requirements.yaml文件中，也可以直接手动的放到charts目录中。推荐的方式是尽量写到requirements.yaml中来动态维护，而不是手动维护。

**requirements.yaml**
```yaml
dependencies:
  - name: apache
    version: 1.2.3
    repository: http://example.com/charts （库的地址必须是实现已经helm repo add 进来的）
  - name: mysql
    version: 3.2.1
    repository: http://another.example.com/charts 
    alias: new-apache (下载的依赖包名就为该字段名称) 可选
    tags：可选
      - front-end
      - subchart1
    condition：subchart1.enabled,global.subchart1.enabled 可选
```
更新了requirements.yaml之后，可以运行命令`helm dependency update`, helm会读取其中的所有chart然后下载到charts/这个目录中。charts目录里会有类似的包：
```
charts/
  apache-1.2.3.tgz
  mysql-3.2.1.tgz
```
可选的字段还有tags和condition，默认heml会下载所有的chart的，但是设置了通过tags和condition可以对chart进行一定程度的过滤。tags和condition设置的参数会在 values.yaml指定value（true or false)

**chart/目录**
可以手动的下载chart到该目录，chart可以是如上的压缩包，也可以是一个chart的目录
```
wordpress:
  Chart.yaml
  requirements.yaml
  # ...
  charts/
    apache/
      Chart.yaml
      # ...
    mysql/
      Chart.yaml
      # ...
```

#### templates 和values yaml
templates目录下定义了一系列的模板文件，而在values.yaml中定义了默认值，在helm install的时候，用户可以指定value来覆盖默认值。

**templates 文件**
符合go语言的[模板包](https://golang.org/pkg/text/template/), 一个例子：
```
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    app.kubernetes.io/managed-by: deis
spec:
  replicas: 1
  selector:
    app.kubernetes.io/name: deis-database
  template:
    metadata:
      labels:
        app.kubernetes.io/name: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{.Values.imageRegistry}}/postgres:{{.Values.dockerTag}}
          imagePullPolicy: {{.Values.pullPolicy}}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{default "minio" .Values.storage}}
```
这个例子模板可以使用value.yaml定义的imageRegistry，dockerTag，pullPolicy，storage来渲染。

除了在value.yaml中定义的之外，还有一些预定于的filed比如Release.Name可以使用。


**value.yaml**
默认helm install 会使用values.yaml 如果由其他自定义file可以使用--values=another.yaml来指定。another.yaml中指定的字段会和values.yaml中指定的字段进行合并，如果字段相同，将会覆盖values.yaml中的字段值。




#### 使用helm管理chart
**Chart 库**
只要是能够server yaml文件以及tar 文件的，并且能响应GET请求的HTTP server都可以被作为一个chart 库
**CHart starter packs**

### Hooks

### chart开发技巧
#### 熟悉模板函数 支持所胡所有的函数在[sprig library](https://godoc.org/github.com/Masterminds/sprig)中。


### Chart Repostiory 指南
官方的chart 库被[kubernters charts](https://github.com/helm/charts)维护，我们可以使用helm来维护自己的repo。

#### 创建chart repo
只要是能处理yaml文件和tar包的http server就可以作为一个repo，所以我们由很多中选择。一个`https://example.com/charts` 的repo形如：
```
charts/
  |
  |- index.yaml
  |
  |- alpine-0.1.2.tgz
  |
  |- alpine-0.1.2.tgz.prov
```
没有要求index.yaml和tgz包必须要在同一个位置，但推荐这么做。

**index.yaml**
包含一些metadata以及Chart.yaml里的内容，在本地可以使用helm repo index命令来生成一个index 文件。
```
apiVersion: v1
entries:
  alpine:
    - created: 2016-10-06T16:23:20.499814565-06:00
      description: Deploy a basic Alpine Linux pod
      digest: 99c76e403d752c84ead610644d4b1c2f2b453a74b921f422b9dcb8a7c8b559cd
      home: https://k8s.io/helm
      name: alpine
      sources:
      - https://github.com/kubernetes/helm
      urls:
      - https://technosophos.github.io/tscharts/alpine-0.2.0.tgz
      version: 0.2.0
    - created: 2016-10-06T16:23:20.499543808-06:00
      description: Deploy a basic Alpine Linux pod
      digest: 515c58e5f79d8b2913a10cb400ebb6fa9c77fe813287afbacf1a0b897cd78727
      home: https://k8s.io/helm
      name: alpine
      sources:
      - https://github.com/kubernetes/helm
      urls:
      - https://technosophos.github.io/tscharts/alpine-0.1.0.tgz
      version: 0.1.0
  nginx:
    - created: 2016-10-06T16:23:20.499543808-06:00
      description: Create a basic nginx HTTP server
      digest: aaff4545f79d8b2913a10cb400ebb6fa9c77fe813287afbacf1a0b897cdffffff
      home: https://k8s.io/helm
      name: nginx
      sources:
      - https://github.com/kubernetes/charts
      urls:
      - https://technosophos.github.io/tscharts/nginx-1.1.0.tgz
      version: 1.1.0
generated: 2016-10-06T16:23:20.499029981-06:00
```
测试的话可以直接使用helm server在本地起一个测试环境。
```
$ helm serve --repo-path ./charts
Regenerating index. This may take a moment.
Now serving you on 127.0.0.1:8879
```

#### hosting chart repo
有很多种方式可选。GoogleCloudStorage，Amazon S3，以及github pages


####维护chart repo


### helm plugin
```
helm plugin install <path|url>
```


### 问题处理

#### 1. 启动tiller后，helm链接不上。
默认tirrel service 的porttype是 CLusterID， 集群外部是无法访问到的。需要姜type类型改为NodePort。
```bash
[root@rdb24 ~]# kubectl edit service/tiller-deploy -n kube-system
service "tiller-deploy" edited
```
修改完之后，会看到 服务的port 44134 映射到了 node节点的31740
```
[root@rdb24 ~]# kubectl get service/tiller-deploy -n kube-system
NAME            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGE
tiller-deploy   NodePort   10.200.119.214   <none>        44134:31740/TCP   5h
```
然后, 在helm节点设置环境变量，即可.运行helm version 检验
```
export HELM_HOST=10.254.10.27:31740
[root@rdb24 ~]# helm version
Client: &version.Version{SemVer:"v2.10.0", GitCommit:"9ad53aac42165a5fadc6c87be0dea6b115f93090", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.10.0", GitCommit:"9ad53aac42165a5fadc6c87be0dea6b115f93090", GitTreeState:"clean"}

```

#### 2. 连接权限问题
报错内容：
```
rror: configmaps is forbidden: User "system:serviceaccount:kube-system:default" cannot list configmaps in the namespace "kube-system"
```
解决：
创建一个新的serviceaccount，赋予cluster-admin的role，并更新到tiller-deploy中
```
[root@rdb24 ~]# kubectl create serviceaccount --namespace kube-system tiller
serviceaccount "tiller" created
[root@rdb24 ~]# kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
clusterrolebinding.rbac.authorization.k8s.io "tiller-cluster-rule" created
[root@rdb24 ~]# kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
deployment.extensions "tiller-deploy" patched
```
