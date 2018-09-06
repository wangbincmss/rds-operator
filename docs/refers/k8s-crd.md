## 创建一个CRD
```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # 名称必须和下面的spec中的字段相对应，名称格式: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # 用于 REST API: /apis/<group>/<version>
  group: stable.example.com
  # 列出该CRD支持的所有版本
  versions:
    - name: v1
      # 该字段enable或者disable次version
      served: true
      # 一个且只有一个version 这个此段必须被设置为true
      storage: true
  # 是集群范围内还是ns范围内
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
```

创建：
```
kubectl create -f resourcedefinition.yaml
```
一个新的ns级别的RESTful API 端点创建在：
```
/apis/stable.example.com/v1/namespaces/*/crontabs/...
```


## 创建一个自定义对象
通过CRD定义了一个CR之后，就可以像对pod那样操作他了
```
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
```
然后创建：
```
kubectl create -f my-crontal.yaml
```
可以使用get操作
```
kubectl get crontab -o yaml
apiVersion: v1
items:
- apiVersion: stable.example.com/v1
  kind: CronTab
  metadata:
    clusterName: ""
    creationTimestamp: 2017-05-31T12:56:35Z
    deletionGracePeriodSeconds: null
    deletionTimestamp: null
    name: my-new-cron-object
    namespace: default
    resourceVersion: "285"
    selfLink: /apis/stable.example.com/v1/namespaces/default/crontabs/my-new-cron-object
    uid: 9423255b-4600-11e7-af6a-28d2447dc82b
  spec:
    cronSpec: '* * * * */5'
    image: my-awesome-cron-image
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

## 删除一个CRD
当删除一个CRD时，server 会卸载对一个的RESTful API 端点，并且删除所有的自定义对象

```
kubectl delete -f resourcedefinition.yaml
kubectl get crontabs
Error from server (NotFound): Unable to list "crontabs": the server could not find the r
```

## 多版本CRD控制
### 指定多版本
CRD中有一个versions的字段，定义多个version的yaml示例如下（只截取versions部分定义）
```
  # list of versions supported by this CustomResourceDefinition
  versions:
  - name: v1beta1
    # Each version can be enabled/disabled by Served flag.
    served: true
    # 多个版本下也只有一个能为true
    storage: true
  - name: v1
    served: trur
    storage: false
```
对应的RESTful API为/apis/example.com/v1beta1 和 /apis/example.com/v1
 **关于版本优先级**
 各个版本的优先级不是按照在CRD中的定义顺序来的，他会根据特定的算法来排序。
 在k8s中，版本命名规则为：v开头，加上数字，加上可选的beta或者alpha，加上数字。例如：v1或者v1beta2 。排序算法如下：
 - 按名称规则来的排在未按名称规则来的之前。
 - 数字部分从大到小排。
 - 带beta或者alpha的排后面
 - beta或者alpha后面的数字根据数字大小从大到小排
 一个排序的示例：
```
- v10
- v2
- v1
- v11beta2
- v10beta3
- v3beta1
- v12alpha1
- v11alpha2
- foo1
- foo10
```

### 读/写/更新 版本控制下的CRD对象
示例说明：
1. storage version是v1beta1, 创建一个对象，他以v1beta1持久化在存储中。
2. 更新了CRD，添加了一个version v1，并将其storage 为true。
3. 在v1beta1读取该对象，可以读到，在v1读取 也可以读到。并且除了version不一样之外，其他都一样。

### 更新一个已存在的对象到一个新版本
比如要从v1beta1更新到v1
1. 设置v1的storage为true。此时的storedVersions为v1beta1，和v1
2. 写一个过程列出所有存在的对象，并且使用相同内容重写。这个步骤强制后端以v1 进行重写对象。
3. 更新CRD status 从storedVersions字段删除v1beta1

## 高级功能
### Finalizers
Finalizers 允许controllers 实现异步的pre-delete hooks.
```
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  finalizers:
  - finalizer.stable.example.com
```
当一个删除请求发送到一个拥有finalizers的对象时，不会真的删除，会设置一个值给metadata.deletionTimestamp字段。设置了之后，在finalizer list中的记录就只能被删除。
当metadata.deletionTimestamp 设置之后，controller去监控finalizers的执行，当所有finalizers都被执行了，这个资源就删除了。
### Validation   1.11版新增功能
可以对CRD的字段设置验证。如果不想开启该功能，启动kube-apiserver的时候添加:
```
--feature-gates=CustomResourceValidation=false
```
示例,spec.conSpec必须满足正则表达式， replicas必须为1-10的整数
```
spec:
  ...
  ...
  validation:
   # openAPIV3Schema is the schema for validating custom objects.
    openAPIV3Schema:
      properties:
        spec:
          properties:
            cronSpec:
              type: string
              pattern: '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
            replicas:
              type: integer
              minimum: 1
              maximum: 10
```

当我们以其他值进行创建时就会报错：
```
le.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * *"
  image: my-awesome-cron-image
  replicas: 15
  
The CronTab "my-new-cron-object" is invalid: []: Invalid value: map[string]interface {}{"apiVersion":"stable.example.com/v1", "kind":"CronTab", "metadata":map[string]interface {}{"name":"my-new-cron-object", "namespace":"default", "deletionTimestamp":interface {}(nil), "deletionGracePeriodSeconds":(*int64)(nil), "creationTimestamp":"2017-09-05T05:20:07Z", "uid":"e14d79e7-91f9-11e7-a598-f0761cb232d1", "selfLink":"", "clusterName":""}, "spec":map[string]interface {}{"cronSpec":"* * * *", "image":"my-awesome-cron-image", "replicas":15}}:
validation failure list:
spec.cronSpec in body should match '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
spec.replicas in body should be less than or equal to 10
```

### Additional printer columns  1.11新功能
可设置当使用kubectl get时 展示的字段
```
additionalPrinterColumns:
    - name: Spec
      type: string
      description: The cron spec defining the interval a CronJob is run
      JSONPath: .spec.cronSpec
    - name: Replicas
      type: integer
      description: The number of jobs launched by the CronJob
      JSONPath: .spec.replicas
    - name: Age
      type: date
      JSONPath: .metadata.creationTimestamp
      
      
kubectl create -f resourcedefinition.yaml
kubectl get crontab my-new-cron-object
# 此时，会显示 SPEC，REPLICAS， AGE三个字段，其中 name字段时默认包含的
  NAME                 SPEC        REPLICAS   AGE
  my-new-cron-object   * * * * *   1          7s
```
**Priority**
每一个字段还包含一个priority字段， =0 表示显示在标准字段，任何>0的数表示 -o wide 时展示

**Type**
```
integer – non-floating-point numbers
number – floating point numbers
string – strings
boolean – true or false
date – rendered differentially as time since this timestamp.
```

**Fromat**
format字段控制 当使用kubectl print它的值时 他的方式。
```
int32
int64
float
double
byte
date
date-time
password
```

### Subresources
CR 支持两种subresource——/status和/scale（From 1.11）,分别对应了kubectl status和kubectl scale两个命令的响应。

该特性默认是开启的，可以通过`--feature-gates=CustomResourceSubresources=false`来关闭该特性。


**Status subresource**
当我们使用kubectl get xxx -o yaml 去获取一个资源对象的时候，通过输出就可以看出来，大概分为三部分：metadata、spec、status。spec可以理解为我们期望这个资源达到一个什么状态，status可以理解为这个资源当前是一个什么状态。而/status 这个subresources的支持就可以让我们对CRD的资源进行关于status的操作.

**Scale subresource**
当scale subresource激活时，/scale就暴露出来了。autoscaling/v1.Scale 成为/scale的实际响应。要enable它，需要再crd中指定三个参数
- StatusReplicasPath：代表Scale.Status.Replicas
  - 必须值
  - 必须在.status之下
  - 如果在CR的StatusReplicasPath没有value，status的replica值默认时0
- SpecReplicaPath: 代表Scale.Spec.Replicas
  - 必须值
  - 必须在.spec之下
  - 如果没有value，GET /scale请求时会报错。
- LabelSelectorPath: 代表Scale.Status.Selector
  - 可选值
  - 必须在.status之下
  - 如果在CR的StatusReplicasPath没有value，值默认时0
  
示例：
```
  subresources:
    # status enables the status subresource.
    status: {}
    # scale enables the scale subresource.
    scale:
      # specReplicasPath defines the JSONPath inside of a custom resource that corresponds to Scale.Spec.Replicas.
      specReplicasPath: .spec.replicas
      # statusReplicasPath defines the JSONPath inside of a custom resource that corresponds to Scale.Status.Replicas.
      statusReplicasPath: .status.replicas
      # labelSelectorPath defines the JSONPath inside of a custom resource that corresponds to Scale.Status.Selector.
      labelSelectorPath: .status.labelSelector
```

### Catagories
categories表示的是一组资源，比如默认的一个catagories就是all。可以使用命令`kubectl get all` 获取添加进all的所有资源对象信息。默认的就是。使用：
```
...
categories:
- all
```

### 一个综合示例：
CRD.yaml
```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: mysqlclusters.stable.rds.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: stable.rds.com
  # list of versions supported by this CustomResourceDefinition
  versions:
    - name: v1
      # Each version can be enabled/disabled by Served flag.
      served: true
      # One and only one version must be marked as the storage version.
      storage: true
    - name: v2alpha1
      served: true
      storage: false
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: mysqlclusters
    # singular name to be used as an alias on the CLI and for display
    singular: mysqlcluster
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: MysqlCluster
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - mc
    categories:
    - all
  validation:
    openAPIV3Schema:
      properties:
        spec:
          properties:
            password:
              type: string
              pattern: '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
            replicas:
              type: integer
              minimum: 1
              maximum: 10
  additionalPrinterColumns:
  - name: Password
    type: string
    description: Root password
    JSONPath: .spec.password
  - name: Replicas
    type: integer
    description: The number of nodes of mysqlcluster.
    JSONPath: .spec.replicas
  - name: Age
    type: date
    JSONPath: .metadata.creationTimestamp
  # subresources describes the subresources for custom resources.
  subresources:
    # status enables the status subresource.
    status: {}
    # scale enables the scale subresource.
    scale:
      # specReplicasPath defines the JSONPath inside of a custom resource that corresponds to Scale.Spec.Replicas.
      specReplicasPath: .spec.replicas
      # statusReplicasPath defines the JSONPath inside of a custom resource that corresponds to Scale.Status.Replicas.
      statusReplicasPath: .status.replicas
      # labelSelectorPath defines the JSONPath inside of a custom resource that corresponds to Scale.Status.Selector.
      labelSelectorPath: .status.labelSelector

```

CR的示例：
```
apiVersion: "stable.rds.com/v1"
kind: MysqlCluster
metadata:
  name: test-mysql-cluster
spec:
  password: '* * * * */5'
  replicas: 3
```
kubectl get
```
# 使用标准名字，输出的字段就是我们在CRD中定义的 PASSWORD REPLICAS AGE字段
[root@rdb24 crd]# kubectl get mysqlclusters
NAME                 PASSWORD      REPLICAS   AGE
test-mysql-cluster   * * * * */5   3          5m

# 使用短名称
[root@rdb24 crd]# kubectl get mc
NAME                 PASSWORD      REPLICAS   AGE
test-mysql-cluster   * * * * */5   3          5m

# 输出完整内容
[root@rdb24 crd]# kubectl get mc -o yaml
apiVersion: v1
items:
- apiVersion: stable.rds.com/v1
  kind: MysqlCluster
  metadata:
    creationTimestamp: 2018-09-06T07:16:20Z
    generation: 1
    name: test-mysql-cluster
    namespace: default
    resourceVersion: "13087253"
    selfLink: /apis/stable.rds.com/v1/namespaces/default/mysqlclusters/test-mysql-cluster
    uid: c0ef7319-b1a4-11e8-bbad-384c4fcc6ffe
  spec:
    password: '* * * * */5'
    replicas: 3
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
  
# 检查是否包含在all的 cagegories中
[root@rdb24 crd]# kubectl get all
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.200.0.1   <none>        443/TCP   147d

NAME                                             PASSWORD      REPLICAS   AGE
mysqlcluster.stable.rds.com/test-mysql-cluster   * * * * */5   3          8m

```
 检查对应的API 接口 
 检查所有的groups 可以看到我们的stable.rds.com已经注册到了apiserver上,包含v1和v2alpa两个版本。
 ```
 https://10.254.10.24:6443/apis
 
   {
      "name": "stable.rds.com",
      "versions": [
        {
          "groupVersion": "stable.rds.com/v1",
          "version": "v1"
        },
        {
          "groupVersion": "stable.rds.com/v2alpha1",
          "version": "v2alpha1"
        }
      ],
      "preferredVersion": {
        "groupVersion": "stable.rds.com/v1",
        "version": "v1"
      }
    },

```

以V1版本为例，检查一下有多少个source。可以看到有三个resource，其中两个是subresource，分别是status和scale
```
https://10.254.10.24:6443/apis/stable.rds.com/v1/

{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "stable.rds.com/v1",
  "resources": [
    {
      "name": "mysqlclusters",
      "singularName": "mysqlcluster",
      "namespaced": true,
      "kind": "MysqlCluster",
      "verbs": [
        "delete",
        "deletecollection",
        "get",
        "list",
        "patch",
        "create",
        "update",
        "watch"
      ],
      "shortNames": [
        "mc"
      ],
      "categories": [
        "all"
      ]
    },
    {
      "name": "mysqlclusters/status",
      "singularName": "",
      "namespaced": true,
      "kind": "MysqlCluster",
      "verbs": [
        "get",
        "patch",
        "update"
      ]
    },
    {
      "name": "mysqlclusters/scale",
      "singularName": "",
      "namespaced": true,
      "group": "autoscaling",
      "version": "v1",
      "kind": "Scale",
      "verbs": [
        "get",
        "patch",
        "update"
      ]
    }
  ]
}
```

资源——mysqlclusters
```
{"apiVersion":"stable.rds.com/v1",
 "items":[{"apiVersion":"stable.rds.com/v1","kind":"MysqlCluster","metadata":{"creationTimestamp":"2018-09-06T07:16:20Z","generation":1,"name":"test-mysql-cluster","namespace":"default","resourceVersion":"13087253","selfLink":"/apis/stable.rds.com/v1/namespaces/default/mysqlclusters/test-mysql-cluster","uid":"c0ef7319-b1a4-11e8-bbad-384c4fcc6ffe"},"spec":{"password":"* * * * */5","replicas":3}}],
 "kind":"MysqlClusterList",
 "metadata":{"continue":"","resourceVersion":"13089393","selfLink":"/apis/stable.rds.com/v1/namespaces/default/mysqlclusters"}}

```
