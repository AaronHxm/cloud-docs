# RBAC概述

前面已经学习一些常用的资源对象的使用，我们知道对于资源对象的操作都是通过 APIServer 进行的，那么集群是怎样知道我们的请求就是合法的请求呢？这个就需要了解 Kubernetes 中另外一个非常重要的知识点了：RBAC（基于角色的权限控制）。

管理员可以通过 Kubernetes API 动态配置策略来启用RBAC，需要在 kube-apiserver 中添加参数--authorization-mode=RBAC，如果使用的 kubeadm 安装的集群那么是默认开启了 RBAC 的，可以通过查看 Master 节点上 apiserver 的静态 Pod 定义文件：



```shell
➜  ~ cat /etc/kubernetes/manifests/kube-apiserver.yaml
...
    - --authorization-mode=Node,RBAC
...
```

如果是二进制的方式搭建的集群，添加这个参数过后，记得要重启 kube-apiserver 服务。

## API 对象

在学习 RBAC 之前，我们还需要再去理解下 Kubernetes 集群中的对象，我们知道，在 Kubernetes 集群中，Kubernetes 对象是我们持久化的实体，就是最终存入 etcd 中的数据，集群中通过这些实体来表示整个集群的状态。前面我们都直接编写的 YAML 文件，通过 kubectl 来提交的资源清单文件，然后创建的对应的资源对象，那么它究竟是如何将我们的 YAML 文件转换成集群中的一个 API 对象的呢？

这个就需要去了解下**声明式 API**的设计，Kubernetes API 是一个以 JSON 为主要序列化方式的 HTTP 服务，除此之外也支持 Protocol Buffers 序列化方式，主要用于集群内部组件间的通信。为了可扩展性，Kubernetes 在不同的 API 路径（比如/api/v1 或者 /apis/batch）下面支持了多个 API 版本，不同的 API 版本意味着不同级别的稳定性和支持：

- Alpha 级别，例如 v1alpha1 默认情况下是被禁用的，可以随时删除对功能的支持，所以要慎用
- Beta 级别，例如 v2beta1 默认情况下是启用的，表示代码已经经过了很好的测试，但是对象的语义可能会在随后的版本中以不兼容的方式更改
- 稳定级别，比如 v1 表示已经是稳定版本了，也会出现在后续的很多版本中。

在 Kubernetes 集群中，一个 API 对象在 Etcd 里的完整资源路径，是由：Group（API 组）、Version（API 版本）和 Resource（API 资源类型）三个部分组成的。通过这样的结构，整个 Kubernetes 里的所有 API 对象，实际上就可以用如下的树形结构表示出来：

![img](https://cdn.nlark.com/yuque/0/2023/png/12365259/1704035750492-cc6712eb-d98d-4294-b0e0-17039b1ece4c.png)

从上图中我们也可以看出 Kubernetes 的 API 对象的组织方式，在顶层，我们可以看到有一个核心组（由于历史原因，是 /api/v1 下的所有内容而不是在 /apis/core/v1 下面）和命名组（路径 /apis/$NAME/$VERSION）和系统范围内的实体，比如 /metrics。我们也可以用下面的命令来查看集群中的 API 组织形式：



```shell
➜  ~ kubectl get --raw /
{
  "paths": [
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    ......
    "/version"
  ]
}
```

比如我们来查看批处理这个操作，在我们当前这个版本中存在1个版本的操作：/apis/batch/v1，暴露了可以查询和操作的不同实体集合，同样我们还是可以通过 kubectl 来查询对应对象下面的数据：



```shell
➜  ~ kubectl get --raw /apis/batch/v1 | python -m json.tool
{
    "apiVersion": "v1",
    "groupVersion": "batch/v1",
    "kind": "APIResourceList",
    "resources": [
        {
            "categories": [
                "all"
            ],
            "kind": "CronJob",
            "name": "cronjobs",
            "namespaced": true,
            "shortNames": [
                "cj"
            ],
            "singularName": "",
            "storageVersionHash": "sd5LIXh4Fjs=",
            "verbs": [
                "create",
                "delete",
                "deletecollection",
                "get",
                "list",
                "patch",
                "update",
                "watch"
            ]
        },
        {
            "kind": "CronJob",
            "name": "cronjobs/status",
            "namespaced": true,
            "singularName": "",
            "verbs": [
                "get",
                "patch",
                "update"
            ]
        },
        {
            "categories": [
                "all"
            ],
            "kind": "Job",
            "name": "jobs",
            "namespaced": true,
            "singularName": "",
            "storageVersionHash": "mudhfqk/qZY=",
            "verbs": [
                "create",
                "delete",
                "deletecollection",
                "get",
                "list",
                "patch",
                "update",
                "watch"
            ]
        },
        {
            "kind": "Job",
            "name": "jobs/status",
            "namespaced": true,
            "singularName": "",
            "verbs": [
                "get",
                "patch",
                "update"
            ]
        }
    ]
}
```

但是这个操作和我们平时操作 HTTP 服务的方式不太一样，这里我们可以通过 kubectl proxy 命令来开启对 apiserver 的访问：



```shell
➜  ~ kubectl proxy
Starting to serve on 127.0.0.1:8001
```

然后重新开启一个新的终端，我们可以通过如下方式来访问批处理的 API 服务：



```shell
➜  ~ curl http://127.0.0.1:8001/apis/batch/v1
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "batch/v1",
  "resources": [
    {
      "name": "jobs",
      "singularName": "",
      "namespaced": true,
      "kind": "Job",
      "verbs": [
        "create",
        "delete",
        "deletecollection",
        "get",
        "list",
        "patch",
        "update",
        "watch"
      ],
      "categories": [
        "all"
      ],
      "storageVersionHash": "mudhfqk/qZY="
    },
    {
      "name": "jobs/status",
      "singularName": "",
      "namespaced": true,
      "kind": "Job",
      "verbs": [
        "get",
        "patch",
        "update"
      ]
    }
  ]
}
```

通常，Kubernetes API 支持通过标准 HTTP POST、PUT、DELETE 和 GET 在指定 PATH 路径上创建、更新、删除和检索操作，并使用 JSON 作为默认的数据交互格式。

比如现在我们要创建一个 Deployment 对象，那么我们的 YAML 文件的声明就需要怎么写：



```shell
apiVersion: apps/v1
kind: Deployment
```

其中 Deployment 就是这个 API 对象的资源类型（Resource），apps 就是它的组（Group），v1 就是它的版本（Version）。API Group、Version 和 资源就唯一定义了一个 HTTP 路径，然后在 kube-apiserver 端对这个 url 进行了监听，然后把对应的请求传递给了对应的控制器进行处理而已，当然在 Kuberentes 中的实现过程是非常复杂的。

## RBAC

上面我们介绍了 Kubernetes 所有资源对象都是模型化的 API 对象，允许执行 CRUD(Create、Read、Update、Delete) 操作(也就是我们常说的增、删、改、查操作)，比如下面的这些资源：

- Pods
- ConfigMaps
- Deployments
- Nodes
- Secrets
- Namespaces
- ......

对于上面这些资源对象的可能存在的操作有：

- create
- get
- delete
- list
- update
- edit
- watch
- exec
- patch

在更上层，这些资源和 API Group 进行关联，比如 Pods 属于 Core API Group，而 Deployements 属于 apps API Group，现在我们要在 Kubernetes 中通过 RBAC 来对资源进行权限管理，除了上面的这些资源和操作以外，我们还需要了解另外几个概念：

- Rule：规则，规则是一组属于不同 API Group 资源上的一组操作的集合
- Role 和 ClusterRole：角色和集群角色，这两个对象都包含上面的 Rules 元素，二者的区别在于，在 Role 中，定义的规则只适用于单个命名空间，也就是和 namespace 关联的，而 ClusterRole 是集群范围内的，因此定义的规则不受命名空间的约束。另外 Role 和 ClusterRole 在Kubernetes 中都被定义为集群内部的 API 资源，和我们前面学习过的 Pod、Deployment 这些对象类似，都是我们集群的资源对象，所以同样的可以使用 YAML 文件来描述，用 kubectl 工具来管理
- Subject：主题，对应集群中尝试操作的对象，集群中定义了3种类型的主题资源：

- - User Account：用户，这是有外部独立服务进行管理的，管理员进行私钥的分配，用户可以使用 KeyStone 或者 Goolge 帐号，甚至一个用户名和密码的文件列表也可以。对于用户的管理集群内部没有一个关联的资源对象，所以用户不能通过集群内部的 API 来进行管理
  - Group：组，这是用来关联多个账户的，集群中有一些默认创建的组，比如 cluster-admin
  - Service Account：服务帐号，通过 Kubernetes API 来管理的一些用户帐号，和 namespace 进行关联的，适用于集群内部运行的应用程序，需要通过 API 来完成权限认证，所以在集群内部进行权限操作，我们都需要使用到 ServiceAccount，这也是我们这节课的重点

- RoleBinding 和 ClusterRoleBinding：角色绑定和集群角色绑定，简单来说就是把声明的 Subject 和我们的 Role 进行绑定的过程（给某个用户绑定上操作的权限），二者的区别也是作用范围的区别：RoleBinding 只会影响到当前 namespace 下面的资源操作权限，而 ClusterRoleBinding 会影响到所有的 namespace。

接下来我们来通过几个简单的示例，来学习下在 Kubernetes 集群中如何使用 RBAC。

### 只能访问某个 namespace 的普通用户

我们想要创建一个 User Account，只能访问 kube-system 这个命名空间，对应的用户信息如下所示：



```shell
username: cnych
group: youdianzhishi
```

#### 创建用户凭证

我们前面已经提到过，Kubernetes 没有 User Account 的 API 对象，不过要创建一个用户帐号的话也是挺简单的，利用管理员分配给你的一个私钥就可以创建了，这个我们可以参考官方文档中的方法，这里我们来使用 OpenSSL 证书来创建一个 User，当然我们也可以使用更简单的 cfssl工具来创建：

给用户 cnych 创建一个私钥，命名成 cnych.key：



```shell
➜  ~ openssl genrsa -out cnych.key 2048
Generating RSA private key, 2048 bit long modulus
..............................................................................+++
..............................................................................................................................................+++
e is 65537 (0x10001)
```

使用我们刚刚创建的私钥创建一个证书签名请求文件：cnych.csr，要注意需要确保在 -subj 参数中指定用户名和组(CN表示用户名，O表示组)：



```plain
➜  ~ openssl req -new -key cnych.key -out cnych.csr -subj "/CN=cnych/O=youdianzhishi"
```

然后找到我们的 Kubernetes 集群的 CA 证书，我们使用的是 kubeadm 安装的集群，CA 相关证书位于 /etc/kubernetes/pki/ 目录下面，如果你是二进制方式搭建的，你应该在最开始搭建集群的时候就已经指定好了 CA 的目录，我们会利用该目录下面的 ca.crt 和 ca.key两个文件来批准上面的证书请求。生成最终的证书文件，我们这里设置证书的有效期为 500 天：



```plain
➜  ~ openssl x509 -req -in cnych.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out cnych.crt -days 500
Signature ok
subject=/CN=cnych/O=youdianzhishi
Getting CA Private Key
```

现在查看我们当前文件夹下面是否生成了一个证书文件：



```plain
➜  ~ ls
cnych.crt  cnych.csr  cnych.key
```

现在我们可以使用刚刚创建的证书文件和私钥文件在集群中创建新的凭证和上下文(Context):



```plain
➜  ~ kubectl config set-credentials cnych --client-certificate=cnych.crt --client-key=cnych.key
User "cnych" set.
```

我们可以看到一个用户 cnych 创建了，然后为这个用户设置新的 Context，我们这里指定特定的一个 namespace：



```plain
➜  ~ kubectl config set-context cnych-context --cluster=kubernetes --namespace=kube-system --user=cnych
Context "cnych-context" created.
```

到这里，我们的用户 cnych 就已经创建成功了，现在我们使用当前的这个配置文件来操作 kubectl 命令的时候，应该会出现错误，因为我们还没有为该用户定义任何操作的权限呢：



```plain
➜  ~ kubectl get pods --context=cnych-context
Error from server (Forbidden): pods is forbidden: User "cnych" cannot list resource "pods" in API group "" in the namespace "kube-system"
```

#### 创建角色

用户创建完成后，接下来就需要给该用户添加操作权限，我们来定义一个 YAML 文件，创建一个允许用户操作 Deployment、Pod、ReplicaSets 的角色，如下定义：



```plain
# cnych-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cnych-role
  namespace: kube-system
rules:
- apiGroups: ["", "apps"]
  resources: ["deployments", "replicasets", "pods"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"] # 也可以使用['*']
```

其中 Pod 属于 core 这个 API Group，在 YAML 中用空字符就可以，而 Deployment 和 ReplicaSet 现在都属于 apps 这个 API Group（如果不知道则可以用 kubectl explain 命令查看），所以 rules 下面的 apiGroups 就综合了这几个资源的 API Group：["", "apps"]，其中 verbs 就是我们上面提到的可以对这些资源对象执行的操作，我们这里需要所有的操作方法，所以我们也可以使用['*']来代替，然后直接创建这个 Role：



```plain
➜  ~ kubectl create -f cnych-role.yaml
role.rbac.authorization.k8s.io/cnych-role created
```

注意这里我们没有使用上面的 cnych-context 这个上下文，因为暂时还木有权限。

#### 创建角色权限绑定

Role 创建完成了，但是很明显现在我们这个 Role 和我们的用户 cnych 还没有任何关系，对吧？这里就需要创建一个 RoleBinding 对象，在 kube-system 这个命名空间下面将上面的 cnych-role 角色和用户 cnych 进行绑定：



```plain
# cnych-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cnych-rolebinding
  namespace: kube-system
subjects:
- kind: User
  name: cnych
  apiGroup: ""
roleRef:
  kind: Role
  name: cnych-role
  apiGroup: rbac.authorization.k8s.io  # 留空字符串也可以，则使用当前的apiGroup
```

上面的 YAML 文件中我们看到了 subjects 字段，这里就是我们上面提到的用来尝试操作集群的对象，这里对应上面的 User 帐号 cnych，使用kubectl 创建上面的资源对象：



```plain
➜  ~ kubectl create -f cnych-rolebinding.yaml
rolebinding.rbac.authorization.k8s.io/cnych-rolebinding created
```

#### 测试

现在我们应该可以上面的 cnych-context 上下文来操作集群了：



```plain
➜  ~ kubectl get pods --context=cnych-context
NAME                              READY   STATUS    RESTARTS        AGE
coredns-7568f67dbd-7std5          1/1     Running   25 (3h5m ago)   29d
coredns-7568f67dbd-wpjpf          1/1     Running   25 (3h5m ago)   29d
etcd-master1                      1/1     Running   31 (3h5m ago)   30d
kube-apiserver-master1            1/1     Running   27 (3h5m ago)   30d
kube-controller-manager-master1   1/1     Running   27 (3h5m ago)   30d
......
➜  ~ kubectl --context=cnych-context get rs,deploy
NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/coredns-7568f67dbd          2         2         2       30d
replicaset.apps/metrics-server-6f667d74b6   0         0         0       9d
replicaset.apps/metrics-server-85499dc4f5   1         1         1       9d

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coredns          2/2     2            2           30d
deployment.apps/metrics-server   1/1     1            1           9d
```

我们可以看到我们使用 kubectl 的使用并没有指定 namespace，这是因为我们我们上面创建这个 Context 的时候就绑定在了 kube-system 这个命名空间下面，如果我们在后面加上一个-n default试看看呢？



```plain
➜  ~ kubectl --context=cnych-context get pods --namespace=default
Error from server (Forbidden): pods is forbidden: User "cnych" cannot list resource "pods" in API group "" in the namespace "default"
```

如果去获取其他的资源对象呢：



```plain
➜  ~ kubectl --context=cnych-context get svc
Error from server (Forbidden): services is forbidden: User "cnych" cannot list resource "services" in API group "" in the namespace "kube-system"
```

我们可以看到没有权限获取，因为我们并没有为当前操作用户指定其他对象资源的访问权限，是符合我们的预期的。这样我们就创建了一个只有单个命名空间访问权限的普通 User 。

### 只能访问某个 namespace 的 ServiceAccount

上面我们创建了一个只能访问某个命名空间下面的**普通用户**，我们前面也提到过 subjects 下面还有一种类型的主题资源：ServiceAccount，现在我们来创建一个集群内部的用户只能操作 kube-system 这个命名空间下面的 pods 和 deployments，首先来创建一个 ServiceAccount 对象：



```plain
➜  ~ kubectl create sa cnych-sa -n kube-system
```

当然我们也可以定义成 YAML 文件的形式来创建：



```plain
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cnych-sa
  namespace: kube-system
```

然后新建一个 Role 对象：(cnych-sa-role.yaml)



```plain
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cnych-sa-role
  namespace: kube-system
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

可以看到我们这里定义的角色没有创建、删除、更新 Pod 的权限，待会我们可以重点测试一下，创建该 Role 对象：



```plain
➜  ~ kubectl apply -f cnych-sa-role.yaml
```

然后创建一个 RoleBinding 对象，将上面的 cnych-sa 和角色 haimaxy-sa-role 进行绑定：(haimaxy-sa-rolebinding.yaml)



```plain
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cnych-sa-rolebinding
  namespace: kube-system
subjects:
- kind: ServiceAccount
  name: cnych-sa
  namespace: kube-system
roleRef:
  kind: Role
  name: cnych-sa-role
  apiGroup: rbac.authorization.k8s.io
```

添加这个资源对象：



```plain
➜  ~ kubectl create -f cnych-sa-rolebinding.yaml
```

然后我们怎么去验证这个 ServiceAccount 呢？我们前面的课程中是不是提到过一个 ServiceAccount 会生成一个 Secret 对象和它进行映射，这个 Secret 里面包含一个 token，我们可以利用这个 token 去登录 Dashboard，然后我们就可以在 Dashboard 中来验证我们的功能是否符合预期了：



```plain
➜  ~ kubectl get secret -n kube-system |grep cnych-sa
cnych-sa-token-nxgqx                  kubernetes.io/service-account-token   3         47m
➜  ~ kubectl get secret cnych-sa-token-nxgqx -o jsonpath={.data.token} -n kube-system |base64 -d
# 会生成一串很长的base64后的字符串
```

使用这里的 token 去 Dashboard 页面进行登录：

![img](https://cdn.nlark.com/yuque/0/2023/png/12365259/1704035750501-1ea2e862-a23b-4ad3-a979-d56f3289154b.png)

我们可以看到上面的提示信息说我们现在使用的这个 ServiceAccount 没有权限获取当前命名空间下面的资源对象，这是因为我们登录进来后默认跳转到 default 命名空间，我们切换到 kube-system 命名空间下面就可以了：

![img](https://cdn.nlark.com/yuque/0/2023/png/12365259/1704035750550-e999dcc8-d416-4dea-aecc-9796acb604b7.png)

我们可以看到可以访问 pod 列表了，但是也会有一些其他额外的提示：events is forbidden: User “system:serviceaccount:kube-system:cnych-sa” cannot list events in the namespace “kube-system”，这是因为当前登录用只被授权了访问 pod 和 deployment 的权限，同样的，访问下deployment看看可以了吗？

同样的，你可以根据自己的需求来对访问用户的权限进行限制，可以自己通过 Role 定义更加细粒度的权限，也可以使用系统内置的一些权限……

### 可以全局访问的 ServiceAccount

刚刚我们创建的 cnych-sa 这个 ServiceAccount 和一个 Role 角色进行绑定的，如果我们现在创建一个新的 ServiceAccount，需要他操作的权限作用于所有的 namespace，这个时候我们就需要使用到 ClusterRole 和 ClusterRoleBinding 这两种资源对象了。同样，首先新建一个 ServiceAcount 对象：(cnych-sa2.yaml)



```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cnych-sa2
  namespace: kube-system
```

创建：



```plain
➜  ~ kubectl create -f cnych-sa2.yaml
```

然后创建一个 ClusterRoleBinding 对象（cnych-clusterolebinding.yaml）:



```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: cnych-sa2-clusterrolebinding
subjects:
- kind: ServiceAccount
  name: cnych-sa2
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

从上面我们可以看到我们没有为这个资源对象声明 namespace，因为这是一个 ClusterRoleBinding 资源对象，是作用于整个集群的，我们也没有单独新建一个 ClusterRole 对象，而是使用的 cluster-admin 这个对象，这是 Kubernetes 集群内置的 ClusterRole 对象，我们可以使用 kubectl get clusterrole 和 kubectl get clusterrolebinding 查看系统内置的一些集群角色和集群角色绑定，这里我们使用的 cluster-admin 这个集群角色是拥有最高权限的集群角色，所以一般需要谨慎使用该集群角色。

创建上面集群角色绑定资源对象，创建完成后同样使用 ServiceAccount 对应的 token 去登录 Dashboard 验证下：



```shell
➜  ~ kubectl create -f cnych-clusterolebinding.yaml
➜  ~ kubectl get secret -n kube-system |grep cnych-sa2
cnych-sa2-token-nxgqx                  kubernetes.io/service-account-token   3         47m
➜  ~ kubectl get secret cnych-sa2-token-nxgqx -o jsonpath={.data.token} -n kube-system |base64 -d
# 会生成一串很长的base64后的字符串
```

![img](https://cdn.nlark.com/yuque/0/2023/png/12365259/1704035750501-189d7281-9f57-4ede-b023-45094499609c.png)

我们在最开始接触到 RBAC 认证的时候，可能不太熟悉，特别是不知道应该怎么去编写 rules 规则，大家可以去分析系统自带的 clusterrole、clusterrolebinding 这些资源对象的编写方法，怎么分析？还是利用 kubectl 的 get、describe、 -o yaml 这些操作，所以 kubectl 最基本的操作用户一定要掌握好。



# RBAC实战

https://kubernetes.io/zh/docs/concepts/security/controlling-access/

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669439416750-c6a8e3c0-24a1-4b74-9c4c-a22717b134a0.png)

NFS的动态供应； Pod；pvc---自动创建pv

k8s会认为每个Pod也可以是操作集群的一个用户。给这个用户会给一个ServiceAccount（服务账号）

权限控制流程：

- 用户携带令牌或者证书给k8s的api-server发送请求要求修改集群资源
- k8s开始认证。认证通过
- k8s查询用户的授权（有哪些权限）
- 用户执行操作。过程中的一些操作（cpu、内存、硬盘、网络等....），利用准入控制来判断是否可以允许这样操作

# RBAC

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669439703320-9b314300-55d8-4ca8-8763-e3b2b4b14a16.png)

什么是RBAC？（基于角色的访问控制）

RBAC API 声明了四种 Kubernetes 对象：Role、ClusterRole、RoleBinding 和 ClusterRoleBinding

Role：基于名称空间的角色。可以操作名称空间下的资源

​	RoleBinding： 来把一个Role。绑定给一个用户

ClusterRole：基于集群的角色。可以操作集群资源

​	ClusterRoleBinding： 来把一个ClusterRole，绑定给一个用户

# ClusterRole与Role

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669452300604-2e0c3155-c7cf-4108-a8d9-5f9966bd13fd.png)

- RBAC 的 *Role* 或 *ClusterRole* 中包含一组代表相关权限的规则。 这些权限是纯粹累加的（不存在拒绝某操作的规则）。
- Role 总是用来在某个[名称空间](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/namespaces/) 内设置访问权限；在你创建 Role 时，你必须指定该 Role 所属的名字空间
- ClusterRole 则是一个集群作用域的资源。这两种资源的名字不同（Role 和 ClusterRole）是因为 Kubernetes 对象要么是名字空间作用域的，要么是集群作用域的， **不可两者兼具。**

我们kubeadm部署的apiserver是容器化部署的。默认没有同步机器时间。

```shell
[root@k8smaster ~]# kubectl get Role
 kubectl get pods --all-namespaces 查看所有的空间
  kubectl api-resources -owide 查看资源
```



## Role

Role 总是用来在某个[名字空间](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/namespaces/)内设置访问权限； 在你创建 Role 时，你必须指定该 Role 所属的名字空间。

```plain
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" 标明 core API 组
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

注意：资源写复数形式

[root@ityehui ~]# kubectl explain role 查看参数

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default  ## 所属的名称空间
  name: pod-reader 
rules: ## 当前角色的规则
- apiGroups: [""] # "" 标明 core API 组
  resources: ["pods"] # 能操作所有的pod
  # resourceNames: ["pod-taint","pod-time"] ## 指定只能操作某个名字的资源
  verbs: ["get", "watch", "list"] # 动词。
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
  # 在 HTTP 层面，用来访问 Secret 对象的资源的名称为 "secrets"
- resources: ["secrets"]
  apiGroups: [""]
  verbs: ["get", "watch", "list"]
- resources: ["configmaps"]
  apiGroups: [""]
  verbs: ["get", "watch", "list"]




# [create delete deletecollection get list patch update watch]
   apiGroups	<[]string>: 默认留空: 不写代表所有
      # deploy:   extendtions/app

   nonResourceURLs	<[]string>： 能访问的url /api/*

   resourceNames	<[]string>: 能操作的白名单（具体Pod、Deploy的名字）

   resources	<[]string>： 指定能操作的资源Pod、Deploy

   verbs	<[]string> -required- ： 操作动作？

   参照kubectl api-resources -owide 的verbs字段。自己挑一些
   [create delete deletecollection get list patch update watch]
 kubectl get role
  kubectl get role -A
  kubectl get ns
  [root@ityehui ~]# kubectl get all -n kubernetes-dashboard 
```

案例实战

```shell
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default  #所属的名称空间
  name: pod-reader
rules: ## 当前角色的规则
  - apiGroups: [""] # "" 标明 core API 组
    resources: ["pods"] # 能操作所有的pod
    # resourceNames: ["pod-taint","pod-time"] ## 指定只能操作某个名字的资源
    verbs: ["get", "watch", "list"] # 动词。
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
  # 在 HTTP 层面，用来访问 Secret 对象的资源的名称为 "secrets"
  - resources: ["secrets"]
    apiGroups: [""]
    verbs: ["get", "watch", "list"]
  - resources: ["configmaps"]
    apiGroups: [""]
    verbs: ["get", "watch", "list"]
[root@ityehui rabc]# kubectl apply -f role.yaml 
role.rbac.authorization.k8s.io/pod-reader created
clusterrole.rbac.authorization.k8s.io/secret-reader created
[root@ityehui rabc]# kubectl get role
NAME         CREATED AT
pod-reader   2024-01-07T15:23:21Z
```



## ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # "namespace" 被忽略，因为 ClusterRoles 不受名字空间限制
  name: secret-reader
rules:
- apiGroups: [""]
  # 在 HTTP 层面，用来访问 Secret 对象的资源的名称为 "secrets"
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

## 常见示例

https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/#role-examples

# RoleBinding、ClusterRoleBinding

# ServiceAccount

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669454372271-d7164c9a-3970-40c1-849e-35217b321a97.png)

```shell
[root@ityehui rabc]# kubectl get ns
#查看命名空间
[root@ityehui rabc]# kubectl get all -n kubernetes-dashboard
#查看dashboard控制台
[root@ityehui rabc]# kubectl get role -n kubernetes-dashboard
#查看详情
[root@ityehui rabc]# kubectl describe  role kubernetes-dashboard -n kubernetes-dashboard
#查看集群资源
[root@ityehui rabc]# kubectl get clusterrole|grep dash
kubernetes-dashboard 
#查看详情
[root@ityehui rabc]# kubectl describe clusterrole kubernetes-dashboard 
#查看账号
 kubectl get serviceaccount -n kubernetes-dashboard
#查看密钥
 kubectl get secrets -n kubernetes-dashboard
 #描述
 kubectl describe secrets kubernetes-dashboard-token-gcxqt -n kubernetes-dashboard
 #角色绑定
 [root@ityehui ~]# kubectl get clusterrolebinding| grep dash
```

![img](https://cdn.nlark.com/yuque/0/2024/png/12365259/1704642436074-2e677e14-2c6a-4afd-b842-84e57e9a7418.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669454392197-1342aeb8-f9ac-46b8-b661-7f9b0a807ece.png)

## 创建ServiceAccount

```shell
# kubectl create serviceaccount --help

kubectl create serviceaccount lfy -n default --dry-run=client -oyaml

kubect get secrets

kubectl describe secret lfy-token-8bzcz

kubectl api-resources -owide|grep namespace 
```

- 每个名称空间都会有自己默认的服务账号

- - 空的服务账号。
  - 每个Pod都会挂载这个默认服务账号。
  - 每个Pod可以自己声明 serviceAccountName： lfy
  - 特殊Pod（比如动态供应等）需要自己创建SA，并绑定相关的集群Role。给Pod挂载。才能操作

集群几个可用的角色

```plain
cluster-admin:  整个集群全部全部权限  *.* ** *
admin: 很多资源的crud，不包括 直接给api-server发送http请求。/api
edit: 所有资源的编辑修改创建等权限
view: 集群的查看权限
```

## 测试基于ServiceAccount的rbac

```shell
#查看yaml文件
kubectl create serviceaccount lfy -n default --dry-run=client -oyaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: lfy
  namespace: default
# ---
# ## 写Role
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: all-global
subjects:  ## 主体
- kind: ServiceAccount
  name: lfy # 'name' 是不区分大小写的
  namespace: default
  apiGroup: ""
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # namespace: default  ## 所属的名称空间
  name: ns-reader 
rules: ## 当前角色的规则
- apiGroups: [""] # "" 标明 core API 组
  resources: ["namespaces"] ## 获取集群的所有名称空间
  verbs: ["get", "watch", "list"] # 动词。
---
## 编写角色和账号的绑定关系
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-ns-global
subjects:  ## 主体
- kind: ServiceAccount
  name: lfy # 'name' 是不区分大小写的
  namespace: default
  apiGroup: ""
roleRef:
  kind: ClusterRole
  name: ns-reader 
  apiGroup: rbac.authorization.k8s.io
[root@ityehui rabc]# kubectl apply -f role-com.yaml
[root@ityehui rabc]# kubectl get serviceaccount
NAME      SECRETS   AGE
default   0         27d
lfy       0         54s
[root@ityehui rabc]#kubectl describe serviceaccount lfy
[root@ityehui rabc]# kubectl get secrets
[ root@k8s - master rbac]# kubectl describe secrets lfy token-8bzcz
```



## 使用命令行

```shell
## role
kubectl create role pod-reader --verb=get --verb=list --verb=watch --resource=pods
kubectl create role pod-reader --verb=get --resource=pods --resource-name=readablepod --resource-name=anotherpod
kubectl create role foo --verb=get,list,watch --resource=replicasets.apps
kubectl create role foo --verb=get,list,watch --resource=pods,pods/status
kubectl create role my-component-lease-holder --verb=get,list,watch,update --resource=lease --resource-name=my-component
#kubectl create clusterrole
kubectl create clusterrole pod-reader --verb=get,list,watch --resource=pods
kubectl create clusterrole pod-reader --verb=get --resource=pods --resource-name=readablepod --resource-name=anotherpod
kubectl create clusterrole foo --verb=get,list,watch --resource=replicasets.apps
kubectl create clusterrole foo --verb=get,list,watch --resource=pods,pods/status
kubectl create clusterrole "foo" --verb=get --non-resource-url=/logs/*
kubectl create clusterrole monitoring --aggregation-rule="rbac.example.com/aggregate-to-monitoring=true"
#kubectl create rolebinding
kubectl create rolebinding bob-admin-binding --clusterrole=admin --user=bob --namespace=acme
kubectl create rolebinding myapp-view-binding --clusterrole=view --serviceaccount=acme:myapp --namespace=acme
kubectl create rolebinding myappnamespace-myapp-view-binding --clusterrole=view --serviceaccount=myappnamespace:myapp --namespace=acme
###kubectl create clusterrolebinding
kubectl create clusterrolebinding myapp-view-binding --clusterrole=view --serviceaccount=acme:myapp
```

#### role-complate

```shell
apiVersion: v1
kind: ServiceAccount
metadata:
  name: lfy
  namespace: default
# ---
# ## 写Role
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: all-global
subjects:  ## 主体
- kind: ServiceAccount
  name: lfy # 'name' 是不区分大小写的
  namespace: default
  apiGroup: ""
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # namespace: default  ## 所属的名称空间
  name: ns-reader 
rules: ## 当前角色的规则
- apiGroups: [""] # "" 标明 core API 组
  resources: ["namespaces"] ## 获取集群的所有名称空间
  verbs: ["get", "watch", "list"] # 动词。
---
## 编写角色和账号的绑定关系
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-ns-global
subjects:  ## 主体
- kind: ServiceAccount
  name: lfy # 'name' 是不区分大小写的
  namespace: default
  apiGroup: ""
roleRef:
  kind: ClusterRole
  name: ns-reader 
  apiGroup: rbac.authorization.k8s.io
```



| apiGroups       | []string | 默认留空: 不写代表所有                                       |
| --------------- | -------- | ------------------------------------------------------------ |
| nonResourceURLs | []string | 能访问的url /api/*                                           |
| resourceNames   | []string | 能操作的白名单（具体Pod、Deploy的名字）                      |
| resources       | []string | 指定能操作的资源Pod、Deploy                                  |
| verbs           | []string | 操作动作 参照kubectl api-resources -owide 的verbs字段。自己挑一些  [create delete deletecollection get list patch update watch] |

## 4、扩展-RestAPI访问k8s集群

步骤：

- 1、创建ServiceAccount、关联相关权限
- 2、使用ServiceAccount对应的Secret中的token作为http访问令牌
- 3、可以参照k8s集群restapi进行操作

```shell
kubectl get C lusterrolebindinglgrep Cluster- admin
```

请求头 Authorization: Bearer 自己的token即可

java也可以这样

```xml
<dependency>
  <groupId>io.kubernetes</groupId>
  <artifactId>client-java</artifactId>
  <version>10.0.0</version>
</dependency>
```


https://kubernetes.io/zh-cn/docs/reference/