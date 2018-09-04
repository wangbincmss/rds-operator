### API Version
- Alpha
  - 版本号中包含alpha（比如V1alpha1）
  - 可见性：位于master版本，包含在某个官方release中，特性默认是disable的，可在apiserver启动时通过flag开启
  - 有可能包含bug
  - 未来可能随时被删除而不做通知
  - 未来可能会对API做不向后兼容的修改而不做通知
  - 只被建议用在测试环境中，不建议用在生产环境中
- Beta
  - 名称中包含beta，比如v1beta2
  - 该功能已经被很好的测试了，被认为是安全的，功能默认是开启。
  - 这个功能不会被删除，但是有可能会修改一些细节。
  - 未来有可能会进行一些不向后兼容的改变，但是一定会通知用户，并且提供版本迁移指导。
  - 该功能可以用在生产环境，但是不建议用在核心业务以及压力比较大的业务，因为beta版本仍然有不兼容更改的可能性。
  - 一旦离开beta版之后，接口基本就不会有变动了。
- Stable
  - vX，X是一个整数（比如v1,v2)
### API Group
API Group 的概念可以更容易的扩展k8s的API。
restful的API 形如：/api/$GROUP_NAME/$VERSION。
在k8s中预定义了几种API Group:
- core group，唯一一个不需要在API中体现$GROUP_NAME的group,形如/api/v1
- 其他命令的Group：apps, batch, extensions, storage.k8s.io，apiextensions.k8s.io

两种方式提供了k8s API的扩展：CRD和aggregator

### Enable API groups
可以通过--runtime-config来enable/disable某些API groups，例如要disable batch/v1：--runtime-config=batch/v1=fales；要enable batch/v2alpha1，--runtime-config=batch/v2alpha1
> 注：如果修改了--runtime-config, 需要重启apiserver和controller-manager。


### Enable resources in the groups
DaemonSets, Deployments, HorizontalPodAutoscalers, Ingress, Jobs and ReplicaSets 这些资源对象默认是开启支持的，通过--runtime-config的设置来开启对其他资源对象的支持例如：`--runtime-config=extensions/v1beta1/deployments=false,extensions/v1beta1/jobs=false` 就是关闭了对其中两种资源对象的API，但是其他资源对象依然是可以用的

