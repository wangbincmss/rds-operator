- V1.8之后RBAC成为稳定版。
- 对应API： rbac.authorization.k8s.io/v1
- 开启RBAC， 在apiserver 中使用 添加--authorization-mode=RBAC

API中定义了四种顶级类型：
#### Role ClusterRole
一个role对应着一个rules的集合。 Role可应用一一个ns中，ClusterRole则是集群可应用于集群范围内。
一个Role的示例：
```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
  ```
  ClusterRole 除了有Role的功能，还能使用在如下的场景：集群范围内的资源（如node），非资源类型endpoint（例如/healthz), 跨ns的资源。
  ClusterRole的例子：
  ```
  kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  # "namespace" omitted since ClusterRoles are not namespaced
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
  ```
  #### RoleBinding/ClusterRolebinding
  示例：
  ```
  # This role binding allows "jane" to read pods in the "default" namespace.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
  ```
  ClusterRoleBinding的例子：
  ```
  # This cluster role binding allows anyone in the "manager" group to read secrets in any namespace.
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
  ```
  ##### Aggregated ClusterRoles
  1.9之后，可以使用aggregationRule参数基于其他ClusterRoles创建新的ClusterRoles
