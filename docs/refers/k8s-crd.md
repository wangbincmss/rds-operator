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

### Catagories
