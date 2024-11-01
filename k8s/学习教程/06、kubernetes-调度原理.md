# ResourceQuota

https://kubernetes.io/zh/docs/concepts/policy/resource-quotas/

## 简介

- 当多个用户或团队共享具有固定节点数目的集群时，人们会担心**有人使用超过其基于公平原则所分配到的资源量。**
- 资源配额是帮助管理员解决这一问题的工具。
- 资源配额，通过 ResourceQuota 对象来定义，**对每个命名空间的资源消耗总量提供限制**。 它可以**限制**命名空间中**某种类型的对象的总数目上限**，**也可以限制命令空间中的 Pod 可以使用的计算资源的总上限**。
- 资源配额的工作方式如下：

- - 不同的团队可以在不同的命名空间下工作，目前这是非约束性的，在未来的版本中可能会通过 ACL (Access Control List 访问控制列表) 来实现强制性约束。
  - **集群管理员可以为每个命名空间创建一个或多个 ResourceQuota 对象。**
  - 当用户在命名空间下创建资源（如 Pod、Service 等）时，Kubernetes 的配额系统会 跟踪集群的资源使用情况，**以确保使用的资源用量不超过 ResourceQuota 中定义的硬性资源限额**。
  - 如果资源创建或者更新请求违反了配额约束，那么该请求会报错（HTTP 403 FORBIDDEN）， 并在消息中给出有可能违反的约束。
  - 如果命名空间下的计算资源 （如 cpu 和 memory）的**配额被启用**，则**用户必须为 这些资源设定请求值（request）和约束值（limit）**，否则配额系统将拒绝 Pod 的创建。 提示: 可使用 LimitRanger 准入控制器来为没有设置计算资源需求的 Pod 设置默认值。

## 实战测试

https://kubernetes.io/zh/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/

## 计算资源配额

| **资源名称**     | **描述**                                                     |
| ---------------- | ------------------------------------------------------------ |
| limits.cpu       | 所有非终止状态的 Pod，其 CPU 限额总量不能超过该值。          |
| limits.memory    | 所有非终止状态的 Pod，其内存限额总量不能超过该值。           |
| requests.cpu     | 所有非终止状态的 Pod，其 CPU 需求总量不能超过该值。          |
| requests.memory  | 所有非终止状态的 Pod，其内存需求总量不能超过该值。           |
| hugepages-<size> | 对于所有非终止状态的 Pod，针对指定尺寸的巨页请求总数不能超过此值。 |
| cpu              | 与 requests.cpu 相同。                                       |
| memory           | 与 requests.memory 相同。                                    |

## 存储资源配额

[https://kubernetes.io/zh/docs/concepts/policy/resource-quotas/#%E5%AD%98%E5%82%A8%E8%B5%84%E6%BA%90%E9%85%8D%E9%A2%9D](https://kubernetes.io/zh/docs/concepts/policy/resource-quotas/#存储资源配额)

| **资源名称**                                                 | **描述**                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| requests.storage                                             | 所有 PVC，存储资源的需求总量不能超过该值。                   |
| persistentvolumeclaims                                       | 在该命名空间中所允许的 [PVC](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) 总量。 |
| <storage-class-name>.storageclass.storage.k8s.io/requests.storage | 在所有与 <storage-class-name> 相关的持久卷申领中，存储请求的总和不能超过该值。 |
| <storage-class-name>.storageclass.storage.k8s.io/persistentvolumeclaims | 在与 storage-class-name 相关的所有持久卷申领中，命名空间中可以存在的[持久卷申领](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)总数。 |

例如，如果一个操作人员针对 gold 存储类型与 bronze 存储类型设置配额， 操作人员可以定义如下配额：

- gold.storageclass.storage.k8s.io/requests.storage: 500Gi
- bronze.storageclass.storage.k8s.io/requests.storage: 100Gi

## 对象数量配额

[https://kubernetes.io/zh/docs/concepts/policy/resource-quotas/#%E5%AF%B9%E8%B1%A1%E6%95%B0%E9%87%8F%E9%85%8D%E9%A2%9D](https://kubernetes.io/zh/docs/concepts/policy/resource-quotas/#对象数量配额)

你可以使用以下语法对所有标准的、命名空间域的资源类型进行配额设置：

- count/<resource>.<group>：用于非核心（core）组的资源
- count/<resource>：用于核心组的资源

这是用户可能希望利用对象计数配额来管理的一组资源示例。

- count/persistentvolumeclaims
- count/services
- count/secrets
- count/configmaps
- count/replicationcontrollers
- count/deployments.apps
- count/replicasets.apps
- count/statefulsets.apps
- count/jobs.batch
- count/cronjobs.batch

对有限的一组资源上实施一般性的对象数量配额也是可能的。 此外，还可以进一步按资源的类型设置其配额。

支持以下类型：

| **资源名称**           | **描述**                                                     |
| ---------------------- | ------------------------------------------------------------ |
| configmaps             | 在该命名空间中允许存在的 ConfigMap 总数上限。                |
| persistentvolumeclaims | 在该命名空间中允许存在的 [PVC](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) 的总数上限。 |
| pods                   | 在该命名空间中允许存在的非终止状态的 Pod 总数上限。Pod 终止状态等价于 Pod 的 .status.phase in (Failed, Succeeded) 为真。 |
| replicationcontrollers | 在该命名空间中允许存在的 ReplicationController 总数上限。    |
| resourcequotas         | 在该命名空间中允许存在的 ResourceQuota 总数上限。            |
| services               | 在该命名空间中允许存在的 Service 总数上限。                  |
| services.loadbalancers | 在该命名空间中允许存在的 LoadBalancer 类型的 Service 总数上限。 |
| services.nodeports     | 在该命名空间中允许存在的 NodePort 类型的 Service 总数上限。  |
| secrets                | 在该命名空间中允许存在的 Secret 总数上限。                   |

## 优先级

```yaml
apiVersion: v1
kind: List  ### 集合   ---
items:
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-high
  spec:
    hard:
      cpu: "1000"
      memory: 200Gi
      pods: "10"
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: ["high"]
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-medium
  spec:
    hard:
      cpu: "10"
      memory: 20Gi
      pods: "10"
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: ["medium"]
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-low
  spec:
    hard:
      cpu: "5"
      memory: 10Gi
      pods: "10"
    scopeSelector:
      matchExpressions:
      - operator : In
        scopeName: PriorityClass
        values: ["low"]
        
---
########################
apiVersion: v1
kind: Pod
metadata:
  name: high-priority
spec:
  containers:
  - name: high-priority
    image: ubuntu
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10;done"]
    resources:
      requests:
        memory: "10Gi"
        cpu: "500m"
      limits:
        memory: "10Gi"
        cpu: "500m"
  priorityClassName: high  ### priorityClass指定的是什么。就优先使用这个配额约束。
```

# LimitRange

https://kubernetes.io/zh/docs/concepts/policy/limit-range/

批量删除

kubectl delete pods my-dep-5b7868d854-6cgxt quota-mem-cpu-demo quota-mem-cpu-demo2 -n hello

## 简介

- 默认情况下， Kubernetes 集群上的容器运行使用的[计算资源](https://kubernetes.io/zh/docs/concepts/configuration/manage-resources-containers/)没有限制。 
- 使用资源配额，集群管理员可以以[名字空间](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/namespaces/)为单位，限制其资源的使用与创建。 
- 在命名空间中，一个 Pod 或 Container 最多能够使用命名空间的资源配额所定义的 CPU 和内存用量。 
- 有人担心，**一个 Pod 或 Container 会垄断所有可用的资源**。 LimitRange 是在命名空间内限制资源分配（给多个 Pod 或 Container）的策略对象。
- 超额指定。配额 1和cpu，1g内存。

- - Pod。 requests: cpu: 1,memory: 1G。这种直接一次性占完
  - 我们需要使用LimitRange限定一个合法范围

- - - 限制每个Pod能写的合理区间

一个 *LimitRange（限制范围）* 对象提供的限制能够做到：

- 在一个命名空间中实施对每个 Pod 或 Container 最小和最大的资源使用量的限制。
- 在一个命名空间中实施对每个 PersistentVolumeClaim 能申请的最小和最大的存储空间大小的限制。
- 在一个命名空间中实施对一种资源的申请值和限制值的比值的控制。
- 设置一个命名空间中对计算资源的默认申请/限制值，并且自动的在运行时注入到多个 Container 中。

## 实战

- [如何配置每个命名空间最小和最大的 CPU 约束](https://kubernetes.io/zh/docs/tasks/administer-cluster/manage-resources/cpu-constraint-namespace/)。
- [如何配置每个命名空间最小和最大的内存约束](https://kubernetes.io/zh/docs/tasks/administer-cluster/manage-resources/memory-constraint-namespace/)。
- [如何配置每个命名空间默认的 CPU 申请值和限制值](https://kubernetes.io/zh/docs/tasks/administer-cluster/manage-resources/cpu-default-namespace/)。
- [如何配置每个命名空间默认的内存申请值和限制值](https://kubernetes.io/zh/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)。
- [如何配置每个命名空间最小和最大存储使用量](https://kubernetes.io/zh/docs/tasks/administer-cluster/limit-storage-consumption/#limitrange-to-limit-requests-for-storage)。

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-min-max-demo-lr
  namespace: hello
spec:
  limits:
  - max:
      cpu: "800m"  ## 最大不超过800m
      memory: "1Gi"  ## Pod不写limit,request，那么Limit、request就会用到默认最大值
    min: 
      cpu: "200m"  ### 起步申请200
      memory: "20m"
    type: Container
apiVersion: v1
kind: Pod
metadata:
  name: quota-mem-cpu-demo2
  namespace: hello
spec:
  containers:
  - name: quota-mem-cpu-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "20Mi"   
        cpu: "900m"   ## 违背了 max.cpu: "800m"
      requests:
        memory: "10Mi"
        cpu: "20m"   ## 20m违背了 min.cpu: "200m"
```

- ResourceQuota：CPU内存都限制了
- LimitRange：只给了CPU的合法区别。

- - 以后Pod只需要写内存的合法区间
  - **LimitRange**都指定范围。Pod可以不用指定，如下，用到默认最大值
  - ![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669423657232-72c3e781-44fa-4870-880e-13ef76ff630b.png)

```yaml
default <map[string]string>: 给limits默认值
   
defaultRequest  <map[string]string>: 给requests默认值的
   
max <map[string]string>: 最大使用量
   
maxLimitRequestRatio  <map[string]string>: 3 
    limit / request <= ratio;
    800/200 = 4 > 3 ## 被拒绝
```



```yaml
min  <map[string]string>: 最小使用量
  
 type <string> -required-: Container、Pod
```



# 调度原理

Pod。scheduler要计算他应该去哪个Node合适。（调度）

nodeSelector ： 指定去哪些Node

## nodeSelector 

nodeSelector 是节点选择约束的最简单推荐形式。nodeSelector 是 PodSpec 的一个字段。 它包含键值对的映射。为了使 pod 可以在某个节点上运行，该节点的标签中 必须包含这里的每个键值对（它也可以具有其他标签）。 最常见的用法的是一对键值对。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd  ## 标签名。每个Node节点可以打标签
```

ingress-nginx：参照

除了你[添加](https://kubernetes.io/zh/docs/concepts/scheduling-eviction/assign-pod-node/#attach-labels-to-node)的标签外，节点还预先填充了一组标准标签。 这些标签有：

- [kubernetes.io/hostname](https://kubernetes.io/zh/docs/reference/kubernetes-api/labels-annotations-taints/#kubernetes-io-hostname)
- [failure-domain.beta.kubernetes.io/zone](https://kubernetes.io/zh/docs/reference/kubernetes-api/labels-annotations-taints/#failure-domainbetakubernetesiozone)
- [failure-domain.beta.kubernetes.io/region](https://kubernetes.io/zh/docs/reference/kubernetes-api/labels-annotations-taints/#failure-domainbetakubernetesioregion)
- [topology.kubernetes.io/zone](https://kubernetes.io/zh/docs/reference/kubernetes-api/labels-annotations-taints/#topologykubernetesiozone)
- [topology.kubernetes.io/region](https://kubernetes.io/zh/docs/reference/kubernetes-api/labels-annotations-taints/#topologykubernetesiozone)
- [beta.kubernetes.io/instance-type](https://kubernetes.io/zh/docs/reference/kubernetes-api/labels-annotations-taints/#beta-kubernetes-io-instance-type)
- [node.kubernetes.io/instance-type](https://kubernetes.io/zh/docs/reference/kubernetes-api/labels-annotations-taints/#nodekubernetesioinstance-type)
- [kubernetes.io/os](https://kubernetes.io/zh/docs/reference/kubernetes-api/labels-annotations-taints/#kubernetes-io-os)
- [kubernetes.io/arch](https://kubernetes.io/zh/docs/reference/kubernetes-api/labels-annotations-taints/#kubernetes-io-arch)

**说明：**

这些标签的值是特定于云供应商的，因此不能保证可靠。 例如，kubernetes.io/hostname 的值在某些环境中可能与节点名称相同， 但在其他环境中可能是一个不同的值。

### 直接不用调度

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-nodename
  labels:
    env: test
spec:
  nodeName: k8s-node1  ## master默认除外  ## scheduler无需工作
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

## Affinity(亲和) and anti-affinity(反亲和)

Pod：到底去哪些机器。

- scheduler 进行自己计算调度
- 某些机器对这些Pod有吸引力。Pod希望scheduler 把他调度到他喜欢的**哪些机器**。

亲和性能设置如下

```yaml
KIND:     Pod
VERSION:  v1

RESOURCE: affinity <Object>

DESCRIPTION:
     If specified, the pod's scheduling constraints

     Affinity is a group of affinity scheduling rules.

FIELDS:
   nodeAffinity	<Object>: 指定亲和的节点（机器）。

   podAffinity	<Object>: 指定亲和的Pod。这个Pod部署到哪里看他亲和的Pod在哪里
     Describes pod affinity scheduling rules (e.g. co-locate this pod in the
     same node, zone, etc. as some other pod(s)).

   podAntiAffinity	<Object>： Pod的反亲和。
     Describes pod anti-affinity scheduling rules (e.g. avoid putting this pod
     in the same node, zone, etc. as some other pod(s)).
# kubectl get nodes --show-labels 查看标签
kubectl get nodes k8s-master -oyaml
```

### Node Affinity （节点亲和）

**nodeSelector 的升级版**。节点亲和概念上类似于 nodeSelector，它使你可以根据节点上的标签来约束 pod 可以调度到哪些节点。

与nodeSelector差异

- 引入运算符：In，NotIn（labelSelector语法）
- 支持枚举label的可能的取值。如 zone in [az1,az2,az2]
- 支持**硬性过滤**和**软性评分**

- - 硬过滤规则支持指定 **多条件之间的逻辑或运算**
  - 软性评分规则支持 **设置条件权重**

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669427209171-26fb2592-02db-4a9e-b708-6f91faa8e977.png)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution: #硬性过滤：排除不具备指定label的node
        nodeSelectorTerms: 
          - matchExpressions:  #所有matchExpressions满足条件才行
              - key: disktype
                operator: In
                values:
                  - ssd
                  - hdd
 ##    matchExpressions	<[]Object> \matchFields <[]Object>
 ## DuringScheduling（调度期间有效）IgnoredDuringExecution（执行期间忽略）、亲和策略与反亲策略只在Pod调度期间有效，执行期间（Pod运行期间）会被忽略。
 ## required指定硬标准、preferred指定软标准
      preferredDuringSchedulingIgnoredDuringExecution:  #软性评分：不具备指定label的node打低分，降低node被选中的几率
        - weight: 1
          preference:
            matchExpressions: 
              - key: disktype
                operator: In
                values:
                  - ssd
  containers:
    - name: with-node-affinity
      image: nginx
      
##注意：如果你修改或删除了 pod 所调度到的节点的标签，pod 不会被删除。换句话说，亲和选择只在 pod 调度期间有效。
```

案例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "busy-affinity-shib"
  namespace: default
  labels:
    app: "busy-affinity-shib"
spec:
  containers:
  - name: busy-affinity-shib
    image: "busybox"
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:  ## 软性分
      - preference: ## 指定我们喜欢的条件  shib
          matchExpressions:
          - key: disk
            values: ["400"]
            operator: Gt
        weight: 90 ## 权重 0-100
      - preference: ## 指定我们喜欢的条件
          matchExpressions:
          - key: gpu
            values: ["40000"]
            operator: Gt  ## node1的gpu不满足这个条件
        weight: 10 ## 权重 0-100

      # requiredDuringSchedulingIgnoredDuringExecution: ## 硬标准
      #   nodeSelectorTerms:
      #   - matchExpressions:
      #       - key: disktype
      #         values: ["ssd","hdd"]
      #         operator: In
              # In（disktype只要是"ssd"或者"hdd"）,
              # NotIn（disktype只要不是"ssd"或者"hdd"）,
              # Exists(disktype只要存在，无论值是什么，value不用写), 
              # DoesNotExist(disktype只要不存在，无论值是什么，value不用写), 
              # Gt（key大于指定的值的节点）, 
              # Lt（key小于指定的值的节点）
查看描述
kubectl describe pod busy-affinitybe
## FailedScheduling  26s   default-scheduler  0/3 nodes are available: 
# 1 node(s) had taint {node-role.kubernetes.io/master: }, 
# that the pod didn't tolerate, （一个节点不能调）
# 2 node(s) didn't match Pod's node affinity/selector.（两个节点不满足）
# kubectl 
# 标签打上以后就分配成功
kubectl label node k8s-node2 disktype=ssd
查看描述 显示成功
kubectl describe pod busy-affinitybe

kubectl get nodes --show-labels

kubectl get nodes 

kubectl label node k8s-node1 cputype=gxp3090
```



### podAffinity/podAntiAffinity

Pod之间的亲和性与反亲和性（inter-pod affinity and anti-affinity）**可以基于已经运行在节点上的 Pod 的标签**（而不是节点的标签）来限定 Pod 可以被调度到哪个节点上。此类规则的表现形式是：

- 当 X 已经运行了一个或者多个满足规则 Y 的 Pod 时，待调度的 Pod 应该（或者不应该 - 反亲和性）在 X 上运行

- - 规则 Y 以 LabelSelector 的形式表述，附带一个可选的名称空间列表与节点不一样，Pod 是在名称空间中的（因此，Pod的标签是在名称空间中的），针对 Pod 的 LabelSelector 必须同时指定对应的名称空间
  - X 是一个拓扑域的概念，例如节点、机柜、云供应商可用区、云供应商地域，等。X 以 topologyKey 的形式表达，该 Key代表了节点上代表拓扑域（topology domain）的一个标签。

Pod 亲和性与反亲和性结合高级别控制器（例如 ReplicaSet、StatefulSet、Deployment 等）一起使用时，可以非常实用。此时可以很容易的将一组工作复杂调度到同一个 topology，例如，同一个节点。 

示例：在一个三节点的集群中，部署一个使用 redis 的 web 应用程序，并期望 web-server 尽可能与 redis 在同一个节点上。

```yaml
#下面是 redis deployment 的 yaml 片段，包含三个副本以及 `app=store` 标签选择器。Deployment 中配置了 `PodAntiAffinity`，确保调度器不会将三个副本调度到一个节点上：
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
      app: store
  replicas: 2
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:  #选Pod的Label
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis:3.2-alpine
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 2
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname" #不能再同一个拓扑网络
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname" # 需要在同一个拓扑网络
      containers:
      - name: web-app
        image: nginx:1.12-alpine
查看机器调度
kubectl get pod -owide
查看标签
kubectl get nodes --show-labels
打标看调度
[root@k8s-master scheduler]kubectl label node k8s-nodel zone=sh-01 node/k8s-node1 
[rootak8s-master scheduler]kubectl label node k8s-node2 zone=sh-01 node/k8s-node2 
kubectl apply -f redis-cache.yaml
kubectl get pods -owide  看node1和node2的结果
```

| **k8s-001**  | **k8s-002** | **k8s-003** |
| ------------ | ----------- | ----------- |
| web-server-1 | webserver-2 | webserver-3 |
| cache-1      | cache-2     | cache-3     |

```yaml
## 
apiVersion: apps/v1
kind: Deployment #DaemonSet+nodeSelector
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
      app: store
  replicas: 2
  template:  ## Pod模板
    metadata:
      labels:
        app: store
    spec:
      containers:
      - name: redis-server
        image: redis:3.2-alpine
      affinity:
        podAntiAffinity: ## 符合以下指定条件的不会被调度过去
          requiredDuringSchedulingIgnoredDuringExecution:  ## 硬标准
            - labelSelector:
                matchExpressions:  ## 如果按照以下标签找到了Pod就不来了
                  - key: app
                    operator: In
                    values:
                    - store
              topologyKey: "kubernetes.io/hostname"  
          ## 拓扑键(划分逻辑区域)。node节点以name为拓扑网络（name相同认为是同一个东西）
          ## 亲和就是都放在这个逻辑区域，反亲和就是必须避免放在同一个逻辑区域

      
## redis很可能部署同一个机器， 100. 5； 每一个机器只希望一个redis
---
## 区域反亲策略
## 
apiVersion: apps/v1
kind: Deployment #DaemonSet+nodeSelector
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
      app: store
  replicas: 2
  template:  ## Pod模板
    metadata:
      labels:
        app: store
    spec:
      containers:
      - name: redis-server
        image: redis:3.2-alpine
      affinity:
        podAntiAffinity: ## 符合以下指定条件的不会被调度过去
          requiredDuringSchedulingIgnoredDuringExecution:  ## 硬标准
            - labelSelector:
                matchExpressions:  ## 如果按照以下标签找到了Pod就不来了
                  - key: app
                    operator: In
                    values:
                    - store
              topologyKey: "zone"  

---


## 部署web
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 2
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        ## 所有策略必须全部判断满足
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            ## 不能再同一个拓扑网络，
            topologyKey: "kubernetes.io/hostname" #不能再同一个拓扑网络
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            ## 必须在同一逻辑分区（拓扑网络）
            topologyKey: "kubernetes.io/hostname" # 需要在同一个拓扑网络
      containers:
      - name: web-app
        image: nginx:1.12-alpine
```

## 污点与容忍

https://kubernetes.io/zh/docs/concepts/scheduling-eviction/taint-and-toleration/

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669439538528-06bb89c8-ee2d-49eb-8733-3f90a100fca4.png)

### 概述

Pod 中存在属性 Node selector / Node affinity，用于将 Pod 指定到合适的节点。

相对的，节点中存在属性 污点 taints，使得节点可以排斥某些 Pod。

污点和容忍（taints and tolerations）成对工作，以确保 Pod 不会被调度到不合适的节点上。

- 可以为节点增加污点（taints，一个节点可以有 0-N 个污点）
- 可以为 Pod 增加容忍（toleration，一个 Pod 可以有 0-N 个容忍）
- 如果节点上存在污点，则该节点不会接受任何不能容忍（tolerate）该污点的 Pod。

污点是打在节点上、机器上的，容忍是打在pod上的

### 向节点添加污点

```shell
kubectl taint nodes node1 key=value:NoSchedule
#该命令为节点 node1 添加了一个污点。污点是一个键值对，在本例中，污点的键为 key，值为 value，污点效果为 NoSchedule。此污点意味着 Kubernetes 不会向该节点调度任何 Pod，除非该 Pod 有一个匹配的容忍（toleration）

kubectl taint nodes 节点名  污点key=污点的值:污点的效果
# 比如matser的污点：有这个污点不能调度
node-role.kubernetes.io/master:NoSchedule

#执行如下命令可以将本例中的污点移除：
kubectl taint nodes node1 key:NoSchedule-
```

污点写法； **k=v:effect  k是污点的名**

haha=hehe:NoExecute ： 节点上的所有。Pod全部被驱逐。

停机维护：

- node1： Pod
- 给node1打个污点。haha=hehe:NoExecute；Pod被赶走。。。。在其他机器拉起

支持的效果 effect 有：

- **NoSchedule**：不调度。不给给我这里调度Pod
- **PreferNoSchedule** 比 NoSchedule 更宽容一些，Kubernetes 将尽量避免将没有匹配容忍的 Pod 调度到该节点上，但是并不是不可以
- **NoExecute** 不能在节点上运行（如果已经运行，将被驱逐）

master节点默认是由一个污点的

![img](https://cdn.nlark.com/yuque/0/2024/png/12365259/1705752618668-6dc84ed4-6adb-453f-86d5-7c709ed4c73f.png)

```shell
查看master 描述污点
kubectl  describe node k8s-master | grep taint
Taints: node-role.kubernets.io/master:NoSchedule
```

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669437392617-773c91dd-4115-4bed-95f6-ad962f3b607a.png)

测试污点

```shell
#给节点打污点
kubectl taint node k8s-node1 haha=heheh:NoSchedule
```

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669437392617-773c91dd-4115-4bed-95f6-ad962f3b607a.png)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "pod-taint-02"
  namespace: default
  labels:
    app: "pod-taint-02"
spec:
  containers:
  - name: pod-taint-02
    image: "busybox"
    command: ["sleep","3600"]
kubectl apply -f pod-taint-02.yaml
kubectl get pod 查看pod全是pending状态
kubectl describe pod od-taint-02  看描述
```

### 向 Pod 添加容忍

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669437392617-773c91dd-4115-4bed-95f6-ad962f3b607a.png)

PodSpec 中有一个 tolerations 字段，可用于向 Pod 添加容忍。下面的两个例子中定义的容忍都可以匹配上例中的污点，包含这些容忍的 Pod 也都可以被调度到 node1 节点上：

```yaml
#容忍1
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
  

#容忍2
tolerations:
- key: "key"
  operator: "Exists"
  effect: "NoSchedule"


#使用容忍
apiVersion: v1
kind: Pod
metadata:
  name: nginx-tolerations
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: "example-key"
    operator: "Exists"
    effect: "NoSchedule"
```

案例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "pod-taint-02"
  namespace: default
  labels:
    app: "pod-taint-02"
spec:
  containers:
  - name: pod-taint-02
    image: "busybox"
    command: ["sleep","3600"]
  tolerations: ## 容忍
  - key: "haha"   #容忍那个污点
    # operator: "Equal"
    # value: "666"
    effect: "NoSchedule"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name:  taint-haha
  namespace: default
  labels:
    app:  taint-haha
spec:
  selector:
    matchLabels:
      app: taint-haha
  replicas: 3
  template:
    metadata:
      labels:
        app:  taint-haha
    spec:
      # initContainers:
        # Init containers are exactly like regular containers, except:
          # - Init containers always run to completion.
          # - Each init container must complete successfully before the next one starts.
      initContainers:
      - name:  init-01 #request-cpu: 300m
        image:  "busybox"
        command: ["sleep","3600"] 
      - name:  init-02 #request-cpu: 100m
        image:  "busybox"
        command: ["sleep","3600"] 
      containers:  ## 哪个启动不了，后面的就不能启动
      - name:  taint-haha  #request-cpu: 200m
        image:  "busybox"
        command: ["sleep","3600"] 
      tolerations: ## 容忍
      - key: "haha"
        operator: "Exists"
        effect: "NoSchedule"
    
```

### 污点与容忍的匹配

当满足如下条件时，Kubernetes 认为容忍和污点匹配：

- 键（key）相同
- 效果（effect）相同
- 污点的operator为：

- - Exists （此时污点中不应该指定 value）
  - 或者 Equal （此时容忍的 value 应与污点的 value 相同）

如果不指定 operator，则其默认为 Equal

特殊情况

存在如下两种特殊情况：

- 容忍中未定义 key 但是定义了 operator 为 Exists，Kubernetes 认为此容忍匹配所有的污点，如下所示：



```yaml
tolerations:
 - operator: "Exists"  ### 啥都能忍
#最终，所有有污点的机器我们都能容忍，都可以调度
```

- 容忍中未定义 effect 但是定义了 key，Kubernetes 认为此容忍匹配所有 effect，如下所示：

```yaml
tolerations:
 - key: "key"
   operator: "Exists"  ## 无论无论效果是什么都能忍


#最终，有这个污点的机器我们可以容忍，pod可以调度和运行
tolerations:
 - key: "key"
   operator: "Exists"  
   effect: "NoExecute"  ## 默认节点有不能执行的污点。节点Pod都会被赶走。Pod只要忍不执行，节点有不执行的污点，Pod也不会被驱逐。可以指定 tolerationSeconds：来说明Pod最多在Node上呆多久



   ## k8s底层发现网络有问题会给这个机器打上
   	## key：“node.kubernetes.io/unreachable”  effect: "NoExecute"。 Pod立马驱动？？
```

**一个节点上可以有多个污点，同时一个 Pod 上可以有多个容忍。**Kubernetes 使用一种类似于过滤器的方法来处理多个节点和容忍：

- 对于节点的所有污点，检查 Pod 上是否有匹配的容忍，如果存在匹配的容忍，则忽略该污点；
- 剩下的不可忽略的污点将对该 Pod 起作用

例如：

- 如果存在至少一个不可忽略的污点带有效果 NoSchedule，则 Kubernetes 不会将 Pod 调度到该节点上
- 如果没有不可忽略的污点带有效果 NoSchedule，但是至少存在一个不可忽略的污点带有效果 PreferNoSchedule，则 Kubernetes 将尽量避免将该 Pod 调度到此节点
- 如果存在至少一个忽略的污点带有效果NoExecute，则：

- - 假设 Pod 已经在该节点上运行，Kubernetes 将从该节点上驱逐（evict）该 Pod
  - 假设 Pod 尚未在该节点上运行，Kubernetes 将不会把 Pod 调度到该节点

案例分析

```yaml
#假设您给一个节点添加了三个污点：
kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 key1=value1:NoExecute
kubectl taint nodes node1 key2=value2:NoSchedule


#同时，有一个 Pod 带有两个容忍
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
```

在这个案例中，Pod 上有两个容忍，匹配了节点的前两个污点，只有节点的第三个污点对该 Pod 来说不可忽略，该污点的效果为 NoSchedule：

- Kubernetes 不会将此 Pod 调度到该节点上
- 如果 Kubernetes 先将 Pod 调度到了该节点，后向该节点添加了第三个污点，则 Pod 将继续在该节点上运行而不会被驱逐（节点上带有 NoExecute 效果的污点已被 Pod 上的第二个容忍匹配，因此被忽略）

通常，在带有效果 NoExecute 的污点被添加到节点时，节点上任何不容忍该污点的 Pod 将被立刻驱逐，而容忍该污点的 Pod 则不会被驱逐。

此外，带有效果 NoExecute 的污点还可以指定一个可选字段 tolerationSeconds，该字段指定了 Pod 在多长时间后被驱逐，例如：

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoExecute"
  tolerationSeconds: 3600
#如果 Pod 已经运行在节点上，再向节点增加此污点时，Pod 将在该节点上继续运行 3600 秒，然后才被驱逐。如果污点在此之间被移除，则 Pod 将不会被驱逐。
```

### 基于污点的驱逐（TaintBasedEviction）

NoExecute 的污点效果，对已经运行在节点上的 Pod 施加如下影响：

- 不容忍该污点的 Pod 将立刻被驱逐
- 容忍该污点的 Pod 在未指定 tolerationSeconds 的情况下，将继续在该节点上运行
- 容忍该污点的 Pod 在指定了 tolerationSeconds 的情况下，将在指定时间超过时从节点上驱逐

tolerationSeconds 字段可以理解为 Pod 容忍该污点的 耐心：

- 超过指定的时间，则达到 Pod 忍耐的极限，Pod 离开所在节点
- 不指定 tolerationSeconds，则认为 Pod 对该污点的容忍是无期限的

自 kubernetes 1.6 以来，kubernetes 的节点控制器在碰到某些特定的条件时，将自动为节点添加污点。这类污点有：

- node.kubernetes.io/not-ready： 节点未就绪。对应着 NodeCondition Ready 为 False 的情况
- node.kubernetes.io/unreachable： 节点不可触达。对应着 NodeCondition Ready 为 Unknown 的情况
- node.kubernetes.io/out-of-disk：节点磁盘空间已满
- node.kubernetes.io/memory-pressure：节点内存吃紧: NoSchedule
- node.kubernetes.io/disk-pressure：节点磁盘吃紧: NoSchedule
- node.kubernetes.io/network-unavailable：节点网络不可用: NoExecute
- node.kubernetes.io/unschedulable：节点不可调度: 
- node.cloudprovider.kubernetes.io/uninitialized：如果 kubelet 是由 "外部" 云服务商启动的，该污点用来标识某个节点当前为不可用的状态。在“云控制器”（cloud-controller-manager）初始化这个节点以后，kubelet将此污点移除

自 kubernetes 1.13 开始，上述特性被默认启用。

例如，某一个包含了大量本地状态的应用，在网络断开时，可能仍然想要在节点上停留比较长的时间，以等待网络能够恢复，而避免从节点上驱逐。此时，该 Pod 的容忍可能如下所示：

```yaml
tolerations:
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 6000 #只在NoExecute这种下有用。污点。
```

如果 Pod 没有 node.kubernetes.io/not-ready 容忍， Kubernetes 将自动为 Pod 添加一个 tolerationSeconds=300 的 node.kubernetes.io/not-ready 容忍。同样的，如果 Pod 没有 node.kubernetes.io/unreachable 容忍，Kubernetes 将自动为 Pod 添加一个 tolerationSeconds=300 的 node.kubernetes.io/unreachable 容忍

这类自动添加的容忍确保了 Pod 在节点发生 not-ready 和 unreachable 问题时，仍然在节点上保留 5 分钟。

**DaemonSet Pod 相对特殊一些**，他们在创建时就添加了不带 tolerationSeconds 的 NoExecute 效果的容忍，适用的污点有：

- node.kubernetes.io/unreachable
- node.kubernetes.io/not-ready

这将确保 DaemonSet Pod 始终不会被驱逐。

自 Kubernetes 1.8 开始，DaemonSet Controller 自动为所有的 DaemonSet Pod 添加如下 NoSchedule 效果的容忍，以防止 DaemonSet 不能正常工作：

- node.kubernetes.io/memory-pressure
- node.kubernetes.io/disk-pressure
- node.kubernetes.io/out-of-disk（只对关键 Pod 生效）
- node.kubernetes.io/unschedulable（不低于 Kubernetes 1.10）
- node.kubernetes.io/network-unavailable（只对 host network 生效）

# 其他

## 拓扑分区约束

```shell
kubectl explain pod.spec.topologySpreadConstraints
```

https://kubernetes.io/zh/docs/concepts/workloads/pods/pod-topology-spread-constraints/

[深入理解拓扑最大倾斜](https://kubernetes.io/blog/2020/05/introducing-podtopologyspread/)

画出自己的机器拓扑分区图。 1000node

更详细的指定拓扑网络的策略；

用来规划和平衡整个集群的资源；maxSkew；



## 资源调度

containers：resources来做资源限定。

- limits：代表运行时的限制量，超了这个就会有OOM，kubelet就会尝试重启
- requests：代表调度时，衡量某个节点是否还有这么多剩余量。request的资源不够，不能调度

问题1：

```shell
Pod： c1 = request: 200m c2 = request: 300m c3 = request: 100m
containers: ## 所有的资源申请量之和。
- name: taint-haha #request-cpu: 200m
image: "busybox"
command: ["sleep","3600"] 
- name: haha-02 # request-cpu: 300m
image: "busybox"
command: ["sleep","3600"] 
- name: haha-03 # request-cpu: 100m
image: "busybox""
command: ["sleep","3600"] 
Node: 
● node1: 300m - cpu
● node2: 500m - cpu
● node3: 700m - cpu ## scheduler会把Pod调度到node3
```

Pod可调度到哪个节点？

问题2：

```shell
Pod： c1 = request: 200m init-c2->request: 300m init-c3->request:100m
Node: 
● node1: 300m 
● node2: 500m
● node3: 700m
```

Pod可以调度到哪个节点

```yaml
initContainers: ## 初始化会结束。前面的执行完执行后面
      - name:  init-01 #request-cpu: 300m
        image:  "busybox"
        command: ["sleep","3600"] 
      - name:  init-02 #request-cpu: 100m
        image:  "busybox"
        command: ["sleep","3600"] 
      containers:  ## Pod启动以后所有容器都是运行的。（总量）
      - name:  taint-haha  #request-cpu: 200m
        image:  "busybox"
        command: ["sleep","3600"] 
node1,2,3都可以执行这个Pod
```

## 命令行

```shell
Basic Commands (Beginner):
create        Create a resource from a file or from stdin.
expose        Take a replication controller, service, deployment or pod and expose xx
run           Run a particular image on the cluster
set           Set specific features on objects

Basic Commands (Intermediate):
explain       Documentation of resources
get           Display one or many resources
edit          Edit a resource on the server
delete        Delete resources by filenames, stdin, resources and names, or by resources

Deploy Commands:
rollout       Manage the rollout of a resource
scale         Set a new size for a Deployment, ReplicaSet or Replication Controller
autoscale     Auto-scale a Deployment, ReplicaSet, StatefulSet, or ReplicationController

Cluster Management Commands:
certificate   Modify certificate resources.
cluster-info  Display cluster info
top           Display Resource (CPU/Memory) usage.
cordon        Mark node as unschedulable
#  打上默认污点 node.kubernetes.io/unschedulable:NoSchedule
# 不可调度只是指，新的Pod不能给当前节点调度，但是不影响其他工作
uncordon      Mark node as schedulable
# 徐晓默认污点 node.kubernetes.io/unschedulable:NoSchedule-
drain         Drain node in preparation for maintenance
## 排空：驱逐节点上的所有Pod资源。（打NoExecute污点和drain都行）
taint         Update the taints on one or more nodes

Troubleshooting and Debugging Commands:
describe      Show details of a specific resource or group of resources
logs          Print the logs for a container in a pod
attach        Attach to a running container
exec          Execute a command in a container
port-forward  Forward one or more local ports to a pod
proxy         Run a proxy to the Kubernetes API server
cp            Copy files and directories to and from containers.
auth          Inspect authorization
debug         Create debugging sessions for troubleshooting workloads and nodes

Advanced Commands:
diff          Diff live version against would-be applied version
apply         Apply a configuration to a resource by filename or stdin
patch         Update field(s) of a resource
replace       Replace a resource by filename or stdin
wait          Experimental: Wait for a specific condition on one or many resources.
kustomize     Build a kustomization target from a directory or URL.

Settings Commands:
label         Update the labels on a resource
annotate      Update the annotations on a resource
completion    Output shell completion code for the specified shell (bash or zsh)

Other Commands:
api-resources Print the supported API resources on the server
api-versions  Print the supported API versions on the server, in the form of "group/version"
config        Modify kubeconfig files
plugin        Provides utilities for interacting with plugins.
version       Print the client and server version information
```



# 调度器

## 调度器介绍

kube-scheduler 是 kubernetes 的核心组件之一，主要负责整个集群资源的调度功能，根据特定的调度算法和策略，将 Pod 调度到最优的工作节点上面去，从而更加合理、更加充分的利用集群的资源，这也是我们选择使用 kubernetes 一个非常重要的理由。如果一门新的技术不能帮助企业节约成本、提供效率，我相信是很难推进的。

### 调度流程

默认情况下，kube-scheduler 提供的默认调度器能够满足我们绝大多数的要求，我们前面和大家接触的示例也基本上用的默认的策略，都可以保证我们的 Pod 可以被分配到资源充足的节点上运行。但是在实际的线上项目中，可能我们自己会比 kubernetes 更加了解我们自己的应用，比如我们希望一个 Pod 只能运行在特定的几个节点上，或者这几个节点只能用来运行特定类型的应用，这就需要我们的调度器能够可控。

kube-scheduler 的主要作用就是根据特定的调度算法和调度策略将 Pod 调度到合适的 Node 节点上去，是一个独立的二进制程序，启动之后会一直监听 API Server，获取到 PodSpec.NodeName 为空的 Pod，对每个 Pod 都会创建一个 binding。

![img](https://cdn.nlark.com/yuque/0/2024/png/12365259/1705820086545-adb2ad4f-0397-4237-93b4-d000ca9201df.png)

这个过程在我们看来好像比较简单，但在实际的生产环境中，需要考虑的问题就有很多了：

- 如何保证全部的节点调度的公平性？要知道并不是所有节点资源配置一定都是一样的
- 如何保证每个节点都能被分配资源？
- 集群资源如何能够被高效利用？
- 集群资源如何才能被最大化使用？
- 如何保证 Pod 调度的性能和效率？
- 用户是否可以根据自己的实际需求定制自己的调度策略？

考虑到实际环境中的各种复杂情况，kubernetes 的调度器采用插件化的形式实现，可以方便用户进行定制或者二次开发，我们可以自定义一个调度器并以插件形式和 kubernetes 进行集成。

kubernetes 调度器的源码位于 kubernetes/pkg/scheduler 中，其中 Scheduler 创建和运行的核心程序，对应的代码在 pkg/scheduler/scheduler.go，如果要查看 kube-scheduler 的入口程序，对应的代码在 cmd/kube-scheduler/scheduler.go。

调度主要分为以下几个部分：

- 首先是**预选过程**，过滤掉不满足条件的节点，这个过程称为 Predicates（过滤）
- 然后是**优选过程**，对通过的节点按照优先级排序，称之为 Priorities（打分）
- 最后从中选择优先级最高的节点，如果中间任何一步骤有错误，就直接返回错误

Predicates 阶段首先遍历全部节点，过滤掉不满足条件的节点，属于强制性规则，这一阶段输出的所有满足要求的节点将被记录并作为第二阶段的输入，如果所有的节点都不满足条件，那么 Pod 将会一直处于 Pending 状态，直到有节点满足条件，在这期间调度器会不断的重试。

所以我们在部署应用的时候，如果发现有 Pod 一直处于 Pending 状态，那么就是没有满足调度条件的节点，这个时候可以去检查下节点资源是否可用。

Priorities 阶段即再次对节点进行筛选，如果有多个节点都满足条件的话，那么系统会按照节点的优先级(priorites)大小对节点进行排序，最后选择优先级最高的节点来部署 Pod 应用。

下面是调度过程的简单示意图：

![img](https://cdn.nlark.com/yuque/0/2024/png/12365259/1705820087582-1d49c3b5-dfe7-474c-8f0a-a2cd36c7412a.png)

更详细的流程是这样的：

- 首先，客户端通过 API Server 的 REST API 或者 kubectl 工具创建 Pod 资源
- API Server 收到用户请求后，存储相关数据到 etcd 数据库中
- 调度器监听 API Server 查看到还未被调度(bind)的 Pod 列表，循环遍历地为每个 Pod 尝试分配节点，这个分配过程就是我们上面提到的两个阶段：
- 预选阶段(Predicates)，过滤节点，调度器用一组规则过滤掉不符合要求的 Node 节点，比如 Pod 设置了资源的 request，那么可用资源比 Pod 需要的资源少的主机显然就会被过滤掉
- 优选阶段(Priorities)，为节点的优先级打分，将上一阶段过滤出来的 Node 列表进行打分，调度器会考虑一些整体的优化策略，比如把 Deployment 控制的多个 Pod 副本尽量分布到不同的主机上，使用最低负载的主机等等策略
- 经过上面的阶段过滤后选择打分最高的 Node 节点和 Pod 进行 binding 操作，然后将结果存储到 etcd 中 最后被选择出来的 Node 节点对应的 kubelet 去执行创建 Pod 的相关操作（当然也是 watch APIServer 发现的）。

从上面我们可以看出调度器的一系列算法由各种插件在调度的不同阶段来完成，下面我们就先来了解下调度框架。

### 调度框架

调度框架定义了一组扩展点，用户可以实现扩展点定义的接口来定义自己的调度逻辑（我们称之为**扩展**），并将扩展注册到扩展点上，调度框架在执行调度工作流时，遇到对应的扩展点时，将调用用户注册的扩展。调度框架在预留扩展点时，都是有特定的目的，有些扩展点上的扩展可以改变调度程序的决策方法，有些扩展点上的扩展只是发送一个通知。

我们知道每当调度一个 Pod 时，都会按照两个过程来执行：调度过程和绑定过程。

调度过程为 Pod 选择一个合适的节点，绑定过程则将调度过程的决策应用到集群中（也就是在被选定的节点上运行 Pod），将调度过程和绑定过程合在一起，称之为**调度上下文（scheduling context）**。需要注意的是调度过程是同步运行的（同一时间点只为一个 Pod 进行调度），绑定过程可异步运行（同一时间点可并发为多个 Pod 执行绑定）。

调度过程和绑定过程遇到如下情况时会中途退出：

- 调度程序认为当前没有该 Pod 的可选节点
- 内部错误

这个时候，该 Pod 将被放回到 **待调度队列**，并等待下次重试。

#### 扩展点（Extension Points）

下图展示了调度框架中的调度上下文及其中的扩展点，一个扩展可以注册多个扩展点，以便可以执行更复杂的有状态的任务。

![img](https://cdn.nlark.com/yuque/0/2024/png/12365259/1705820086337-f560cfae-b266-4626-a527-7be8fad09e71.png)

1. QueueSort 扩展用于对 Pod 的待调度队列进行排序，以决定先调度哪个 Pod，QueueSort 扩展本质上只需要实现一个方法 Less(Pod1, Pod2) 用于比较两个 Pod 谁更优先获得调度即可，同一时间点只能有一个 QueueSort 插件生效。
2. Pre-filter 扩展用于对 Pod 的信息进行预处理，或者检查一些集群或 Pod 必须满足的前提条件，如果 pre-filter 返回了 error，则调度过程终止。
3. Filter 扩展用于排除那些不能运行该 Pod 的节点，对于每一个节点，调度器将按顺序执行 filter 扩展；如果任何一个 filter 将节点标记为不可选，则余下的 filter 扩展将不会被执行。调度器可以同时对多个节点执行 filter 扩展。
4. Post-filter 是一个通知类型的扩展点，调用该扩展的参数是 filter 阶段结束后被筛选为**可选节点**的节点列表，可以在扩展中使用这些信息更新内部状态，或者产生日志或 metrics 信息。
5. Scoring 扩展用于为所有可选节点进行打分，调度器将针对每一个节点调用 Soring 扩展，评分结果是一个范围内的整数。在 normalize scoring 阶段，调度器将会把每个 scoring 扩展对具体某个节点的评分结果和该扩展的权重合并起来，作为最终评分结果。
6. Normalize scoring 扩展在调度器对节点进行最终排序之前修改每个节点的评分结果，注册到该扩展点的扩展在被调用时，将获得同一个插件中的 scoring 扩展的评分结果作为参数，调度框架每执行一次调度，都将调用所有插件中的一个 normalize scoring 扩展一次。
7. Reserve 是一个通知性质的扩展点，有状态的插件可以使用该扩展点来获得节点上为 Pod 预留的资源，该事件发生在调度器将 Pod 绑定到节点之前，目的是避免调度器在等待 Pod 与节点绑定的过程中调度新的 Pod 到节点上时，发生实际使用资源超出可用资源的情况（因为绑定 Pod 到节点上是异步发生的）。这是调度过程的最后一个步骤，Pod 进入 reserved 状态以后，要么在绑定失败时触发 Unreserve 扩展，要么在绑定成功时，由 Post-bind 扩展结束绑定过程。
8. Permit 扩展用于阻止或者延迟 Pod 与节点的绑定。Permit 扩展可以做下面三件事中的一项：
9. approve（批准）：当所有的 permit 扩展都 approve 了 Pod 与节点的绑定，调度器将继续执行绑定过程
10. deny（拒绝）：如果任何一个 permit 扩展 deny 了 Pod 与节点的绑定，Pod 将被放回到待调度队列，此时将触发 Unreserve 扩展
11. wait（等待）：如果一个 permit 扩展返回了 wait，则 Pod 将保持在 permit 阶段，直到被其他扩展 approve，如果超时事件发生，wait 状态变成 deny，Pod 将被放回到待调度队列，此时将触发 Unreserve 扩展
12. Pre-bind 扩展用于在 Pod 绑定之前执行某些逻辑。例如，pre-bind 扩展可以将一个基于网络的数据卷挂载到节点上，以便 Pod 可以使用。如果任何一个 pre-bind 扩展返回错误，Pod 将被放回到待调度队列，此时将触发 Unreserve 扩展。
13. Bind 扩展用于将 Pod 绑定到节点上：

- - 只有所有的 pre-bind 扩展都成功执行了，bind 扩展才会执行
  - 调度框架按照 bind 扩展注册的顺序逐个调用 bind 扩展
  - 具体某个 bind 扩展可以选择处理或者不处理该 Pod
  - 如果某个 bind 扩展处理了该 Pod 与节点的绑定，余下的 bind 扩展将被忽略

1. Post-bind 是一个通知性质的扩展：

- - Post-bind 扩展在 Pod 成功绑定到节点上之后被动调用
  - Post-bind 扩展是绑定过程的最后一个步骤，可以用来执行资源清理的动作

1. Unreserve 是一个通知性质的扩展，如果为 Pod 预留了资源，Pod 又在被绑定过程中被拒绝绑定，则 unreserve 扩展将被调用。Unreserve 扩展应该释放已经为 Pod 预留的节点上的计算资源。在一个插件中，reserve 扩展和 unreserve 扩展应该成对出现。



对于调度框架插件的启用或者禁用，我们可以使用安装集群时的 [KubeSchedulerConfiguration](https://godoc.org/k8s.io/kubernetes/pkg/scheduler/apis/config#KubeSchedulerConfiguration) 资源对象来进行配置。下面的例子中的配置启用了一个实现了 reserve 和 preBind 扩展点的插件，并且禁用了另外一个插件，同时为插件 foo 提供了一些配置信息：



```yaml
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration

---
plugins:
  reserve:
    enabled:
      - name: foo
      - name: bar
    disabled:
      - name: baz
  preBind:
    enabled:
      - name: foo
    disabled:
      - name: baz

pluginConfig:
  - name: foo
    args: >
      foo插件可以解析的任意内容
```

扩展的调用顺序如下：

- 如果某个扩展点没有配置对应的扩展，调度框架将使用默认插件中的扩展
- 如果为某个扩展点配置且激活了扩展，则调度框架将先调用默认插件的扩展，再调用配置中的扩展
- 默认插件的扩展始终被最先调用，然后按照 KubeSchedulerConfiguration 中扩展的激活 enabled 顺序逐个调用扩展点的扩展
- 可以先禁用默认插件的扩展，然后在 enabled 列表中的某个位置激活默认插件的扩展，这种做法可以改变默认插件的扩展被调用时的顺序

假设默认插件 foo 实现了 reserve 扩展点，此时我们要添加一个插件 bar，想要在 foo 之前被调用，则应该先禁用 foo 再按照 bar foo 的顺序激活。示例配置如下所示：



```yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration

---
plugins:
  reserve:
    enabled:
      - name: bar
      - name: foo
    disabled:
      - name: foo
```

在源码目录 pkg/scheduler/framework/plugins/examples 中有几个示范插件，我们可以参照其实现方式。

#### 示例

其实要实现一个调度框架的插件，并不难，我们只要实现对应的扩展点，然后将插件注册到调度器中即可，下面是默认调度器在初始化的时候注册的插件：



```go
// pkg/scheduler/algorithmprovider/registry.go
func NewRegistry() Registry {
    return Registry{
        // FactoryMap:
        // New plugins are registered here.
        // example:
        // {
        //  stateful_plugin.Name: stateful.NewStatefulMultipointExample,
        //  fooplugin.Name: fooplugin.New,
        // }
    }
}
```

但是可以看到默认并没有注册一些插件，所以要想让调度器能够识别我们的插件代码，就需要自己来实现一个调度器了，当然这个调度器我们完全没必要完全自己实现，直接调用默认的调度器，然后在上面的 NewRegistry() 函数中将我们的插件注册进去即可。在 kube-scheduler 的源码文件 kubernetes/cmd/kube-scheduler/app/server.go 中有一个 NewSchedulerCommand 入口函数，其中的参数是一个类型为 Option 的列表，而这个 Option 恰好就是一个插件配置的定义：



```go
// Option configures a framework.Registry.
type Option func(framework.Registry) error

// NewSchedulerCommand creates a *cobra.Command object with default parameters and registryOptions
func NewSchedulerCommand(registryOptions ...Option) *cobra.Command {
    ......
}
```

所以我们完全就可以直接调用这个函数来作为我们的函数入口，并且传入我们自己实现的插件作为参数即可，而且该文件下面还有一个名为 WithPlugin 的函数可以来创建一个 Option 实例：



```go
// WithPlugin creates an Option based on plugin name and factory.
func WithPlugin(name string, factory framework.PluginFactory) Option {
    return func(registry framework.Registry) error {
        return registry.Register(name, factory)
    }
}
```

所以最终我们的入口函数如下所示：



```go
func main() {
    rand.Seed(time.Now().UTC().UnixNano())

    command := app.NewSchedulerCommand(
        app.WithPlugin(sample.Name, sample.New),
    )

    logs.InitLogs()
    defer logs.FlushLogs()

    if err := command.Execute(); err != nil {
        _, _ = fmt.Fprintf(os.Stderr, "%v\n", err)
        os.Exit(1)
    }

}
```

其中 app.WithPlugin(sample.Name, sample.New) 就是我们接下来要实现的插件，从 WithPlugin 函数的参数也可以看出我们这里的 sample.New 必须是一个 framework.PluginFactory 类型的值，而 PluginFactory 的定义就是一个函数：



```go
type PluginFactory = func(configuration *runtime.Unknown, f FrameworkHandle) (Plugin, error)
```

所以 sample.New 实际上就是上面的这个函数，在这个函数中我们可以获取到插件中的一些数据然后进行逻辑处理即可，插件实现如下所示，我们这里只是简单获取下数据打印日志，如果你有实际需求的可以根据获取的数据就行处理即可，我们这里只是实现了 PreFilter、Filter、PreBind 三个扩展点，其他的可以用同样的方式来扩展即可：



```go
// 插件名称
const Name = "sample-plugin"

type Args struct {
    FavoriteColor  string `json:"favorite_color,omitempty"`
    FavoriteNumber int    `json:"favorite_number,omitempty"`
    ThanksTo       string `json:"thanks_to,omitempty"`
}

type Sample struct {
    args   *Args
    handle framework.FrameworkHandle
}

func (s *Sample) Name() string {
    return Name
}

func (s *Sample) PreFilter(pc *framework.PluginContext, pod *v1.Pod) *framework.Status {
    klog.V(3).Infof("prefilter pod: %v", pod.Name)
    return framework.NewStatus(framework.Success, "")
}

func (s *Sample) Filter(pc *framework.PluginContext, pod *v1.Pod, nodeName string) *framework.Status {
    klog.V(3).Infof("filter pod: %v, node: %v", pod.Name, nodeName)
    return framework.NewStatus(framework.Success, "")
}

func (s *Sample) PreBind(pc *framework.PluginContext, pod *v1.Pod, nodeName string) *framework.Status {
    if nodeInfo, ok := s.handle.NodeInfoSnapshot().NodeInfoMap[nodeName]; !ok {
        return framework.NewStatus(framework.Error, fmt.Sprintf("prebind get node info error: %+v", nodeName))
    } else {
        klog.V(3).Infof("prebind node info: %+v", nodeInfo.Node())
        return framework.NewStatus(framework.Success, "")
    }
}

//type PluginFactory = func(configuration *runtime.Unknown, f FrameworkHandle) (Plugin, error)
func New(configuration *runtime.Unknown, f framework.FrameworkHandle) (framework.Plugin, error) {
    args := &Args{}
    if err := framework.DecodeInto(configuration, args); err != nil {
        return nil, err
    }
    klog.V(3).Infof("get plugin config args: %+v", args)
    return &Sample{
        args: args,
        handle: f,
    }, nil
}
```

完整代码可以前往仓库 https://github.com/cnych/sample-scheduler-framework 获取。

实现完成后，编译打包成镜像即可，然后我们就可以当成普通的应用用一个 Deployment 控制器来部署即可，由于我们需要去获取集群中的一些资源对象，所以当然需要申请 RBAC 权限，然后同样通过 --config 参数来配置我们的调度器，同样还是使用一个 KubeSchedulerConfiguration 资源对象配置，可以通过 plugins 来启用或者禁用我们实现的插件，也可以通过 pluginConfig 来传递一些参数值给插件：



```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: sample-scheduler-clusterrole
rules:
  - apiGroups:
      - ""
    resources:
      - endpoints
      - events
    verbs:
      - create
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - delete
      - get
      - list
      - watch
      - update
  - apiGroups:
      - ""
    resources:
      - bindings
      - pods/binding
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - pods/status
    verbs:
      - patch
      - update
  - apiGroups:
      - ""
    resources:
      - replicationcontrollers
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - apps
      - extensions
    resources:
      - replicasets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - apps
    resources:
      - statefulsets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - policy
    resources:
      - poddisruptionbudgets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - persistentvolumeclaims
      - persistentvolumes
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "storage.k8s.io"
    resources:
      - storageclasses
      - csinodes
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "coordination.k8s.io"
    resources:
      - leases
    verbs:
      - create
      - get
      - list
      - update
  - apiGroups:
      - "events.k8s.io"
    resources:
      - events
    verbs:
      - create
      - patch
      - update
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sample-scheduler-sa
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: sample-scheduler-clusterrolebinding
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: sample-scheduler-clusterrole
subjects:
  - kind: ServiceAccount
    name: sample-scheduler-sa
    namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: scheduler-config
  namespace: kube-system
data:
  scheduler-config.yaml: |
    apiVersion: kubescheduler.config.k8s.io/v1beta1
    kind: KubeSchedulerConfiguration
    leaderElection:
      leaderElect: true
      leaseDuration: 15s
      renewDeadline: 10s
      resourceLock: endpointsleases
      resourceName: sample-scheduler
      resourceNamespace: kube-system
      retryPeriod: 2s
    profiles:
      - schedulerName: sample-scheduler
        plugins:
          preFilter:
            enabled:
              - name: "sample-plugin"
          filter:
            enabled:
              - name: "sample-plugin"
        pluginConfig:
          - name: sample-plugin
            args:  # runtime.Object
              favorColor: "#326CE5"
              favorNumber: 7
              thanksTo: "Kubernetes"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-scheduler
  namespace: kube-system
  labels:
    component: sample-scheduler
spec:
  selector:
    matchLabels:
      component: sample-scheduler
  template:
    metadata:
      labels:
        component: sample-scheduler
    spec:
      serviceAccountName: sample-scheduler-sa
      priorityClassName: system-cluster-critical
      volumes:
        - name: scheduler-config
          configMap:
            name: scheduler-config
      containers:
        - name: scheduler
          image: cnych/sample-scheduler:v0.2.4
          imagePullPolicy: IfNotPresent
          command:
            - sample-scheduler
            - --config=/etc/kubernetes/scheduler-config.yaml
            - --v=3
          volumeMounts:
            - name: scheduler-config
              mountPath: /etc/kubernetes
#          livenessProbe:
#            httpGet:
#              path: /healthz
#              port: 10251
#            initialDelaySeconds: 15
#          readinessProbe:
#            httpGet:
#              path: /healthz
#              port: 10251
```

直接部署上面的资源对象即可，这样我们就部署了一个名为 sample-scheduler 的调度器了，接下来我们可以部署一个应用来使用这个调度器进行调度：



```plain
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-scheduler
spec:
  selector:
    matchLabels:
      app: test-scheduler
  template:
    metadata:
      labels:
        app: test-scheduler
    spec:
      schedulerName: sample-scheduler # 指定使用的调度器，不指定使用默认的default-scheduler
      containers:
        - image: nginx:1.7.9
          imagePullPolicy: IfNotPresent
          name: nginx
          ports:
            - containerPort: 80
```

这里需要注意的是我们现在手动指定了一个 schedulerName 的字段，将其设置成上面我们自定义的调度器名称 sample-scheduler。

我们直接创建这个资源对象，创建完成后查看我们自定义调度器的日志信息：



```plain
➜ kubectl get pods -n kube-system -l component=sample-scheduler
NAME                               READY   STATUS    RESTARTS   AGE
sample-scheduler-896658cd7-k7vcl   1/1     Running   0          57s
➜ kubectl logs -f sample-scheduler-896658cd7-k7vcl -n kube-system
I0114 09:14:18.878613       1 eventhandlers.go:173] add event for unscheduled pod default/test-scheduler-6486fd49fc-zjhcx
I0114 09:14:18.878670       1 scheduler.go:464] Attempting to schedule pod: default/test-scheduler-6486fd49fc-zjhcx
I0114 09:14:18.878706       1 sample.go:77] "Start PreFilter Pod" pod="test-scheduler-6486fd49fc-zjhcx"
I0114 09:14:18.878802       1 sample.go:93] "Start Filter Pod" pod="test-scheduler-6486fd49fc-zjhcx" node="node2" preFilterState=&{Resource:{MilliCPU:0 Memory:0 EphemeralStorage:0 AllowedPodNumber:0 ScalarResources:map[]}}
I0114 09:14:18.878835       1 sample.go:93] "Start Filter Pod" pod="test-scheduler-6486fd49fc-zjhcx" node="node1" preFilterState=&{Resource:{MilliCPU:0 Memory:0 EphemeralStorage:0 AllowedPodNumber:0 ScalarResources:map[]}}
I0114 09:14:18.879043       1 default_binder.go:51] Attempting to bind default/test-scheduler-6486fd49fc-zjhcx to node1
I0114 09:14:18.886360       1 scheduler.go:609] "Successfully bound pod to node" pod="default/test-scheduler-6486fd49fc-zjhcx" node="node1" evaluatedNodes=3 feasibleNodes=2
I0114 09:14:18.887426       1 eventhandlers.go:205] delete event for unscheduled pod default/test-scheduler-6486fd49fc-zjhcx
I0114 09:14:18.887475       1 eventhandlers.go:225] add event for scheduled pod default/test-scheduler-6486fd49fc-zjhcx
```

可以看到当我们创建完 Pod 后，在我们自定义的调度器中就出现了对应的日志，并且在我们定义的扩展点上面都出现了对应的日志，证明我们的示例成功了，也可以通过查看 Pod 的 schedulerName 来验证：



```plain
➜ kubectl get pods
NAME                              READY   STATUS    RESTARTS       AGE
test-scheduler-6486fd49fc-zjhcx   1/1     Running   0              35s
➜ kubectl get pod test-scheduler-6486fd49fc-zjhcx -o yaml
......
restartPolicy: Always
schedulerName: sample-scheduler
securityContext: {}
serviceAccount: default
......
```

从 Kubernetes v1.17 版本开始，Scheduler Framework 内置的预选和优选函数已经全部插件化，所以要扩展调度器我们应该掌握并理解调度框架这种方式。

### 调度器调优

作为 kubernetes 集群的默认调度器，kube-scheduler 主要负责将 Pod 调度到集群的 Node 上。在一个集群中，满足一个 Pod 调度请求的所有节点称之为 可调度 Node，调度器先在集群中找到一个 Pod 的可调度 Node，然后根据一系列函数对这些可调度 Node 进行打分，之后选出其中得分最高的 Node 来运行 Pod，最后，调度器将这个调度决定告知 kube-apiserver，这个过程叫做**绑定**。

在 Kubernetes 1.12 版本之前，kube-scheduler 会检查集群中所有节点的可调度性，并且给可调度节点打分。Kubernetes 1.12 版本添加了一个新的功能，允许调度器在找到一定数量的可调度节点之后就停止继续寻找可调度节点。该功能能提高调度器在大规模集群下的调度性能，这个数值是集群规模的百分比，这个百分比通过 percentageOfNodesToScore 参数来进行配置，其值的范围在 1 到 100 之间，最大值就是 100%，如果设置为 0 就代表没有提供这个参数配置。Kubernetes 1.14 版本又加入了一个特性，在该参数没有被用户配置的情况下，调度器会根据集群的规模自动设置一个集群比例，然后通过这个比例筛选一定数量的可调度节点进入打分阶段。该特性使用**线性公式**计算出集群比例，比如 100 个节点的集群下会取 50%，在 5000 节点的集群下取 10%，这个自动设置的参数的最低值是 5%，换句话说，调度器至少会对集群中 5% 的节点进行打分，除非用户将该参数设置的低于 5。

**注意**

当集群中的可调度节点少于 50 个时，调度器仍然会去检查所有节点，因为可调度节点太少，不足以停止调度器最初的过滤选择。如果我们想要关掉这个范围参数，可以将 percentageOfNodesToScore 值设置成 100。

percentageOfNodesToScore 的值必须在 1 到 100 之间，而且其默认值是通过集群的规模计算得来的，另外 **50** 个 Node 的数值是硬编码在程序里面的，设置这个值的作用在于：**当集群的规模是数百个节点并且 percentageOfNodesToScore 参数设置的过低的时候，调度器筛选到的可调度节点数目基本不会受到该参数影响**。当集群规模较小时，这个设置对调度器性能提升并不明显，但是在超过 1000 个 Node 的集群中，将调优参数设置为一个较低的值可以很明显的提升调度器性能。

不过值得注意的是，该参数设置后可能会导致只有集群中少数节点被选为可调度节点，很多 Node 都没有进入到打分阶段，这样就会造成一种后果，一个本来可以在打分阶段得分很高的 Node 甚至都不能进入打分阶段，由于这个原因，所以这个参数不应该被设置成一个很低的值，通常的做法是不会将这个参数的值设置的低于 10，很低的参数值一般在调度器的吞吐量很高且对 Node 的打分不重要的情况下才使用。换句话说，只有当你更倾向于在可调度节点中任意选择一个 Node 来运行这个 Pod 时，才使用很低的参数设置。

如果你的集群规模只有数百个节点或者更少，实际上并不推荐你将这个参数设置得比默认值更低，因为这种情况下不太会有效的提高调度器性能。

### 优先级调度

与前面所讲的调度优选策略中的优先级（Priorities）不同，前面所讲的优先级指的是节点优先级，而我们这里所说的优先级指的是 Pod 的优先级，高优先级的 Pod 会优先被调度，或者在资源不足的情况牺牲低优先级的 Pod，以便于重要的 Pod 能够得到资源部署。

要定义 Pod 优先级，就需要先定义 PriorityClass 对象，该对象没有 Namespace 的限制：



```yaml
apiVersion: v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."
```

其中：

- value 为 32 位整数的优先级，该值越大，优先级越高
- globalDefault 用于未配置 PriorityClassName 的 Pod，整个集群中应该只有一个 PriorityClass 将其设置为 true

然后通过在 Pod 的 spec.priorityClassName 中指定已定义的 PriorityClass 名称即可：



```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx
      imagePullPolicy: IfNotPresent
  priorityClassName: high-priority
```

另外一个值得注意的是当节点没有足够的资源供调度器调度 Pod，导致 Pod 处于 pending 时，抢占（preemption）逻辑就会被触发，抢占会尝试从一个节点删除低优先级的 Pod，从而释放资源使高优先级的 Pod 得到节点资源进行部署。



## 亲和性调度

一般情况下我们部署的 Pod 是通过集群的自动调度策略来选择节点的，默认情况下调度器考虑的是资源足够，并且负载尽量平均，但是有的时候我们需要能够更加细粒度的去控制 Pod 的调度，比如我们希望一些机器学习的应用只跑在有 GPU 的节点上；但是有的时候我们的服务之间交流比较频繁，又希望能够将这服务的 Pod 都调度到同一个的节点上。这就需要使用一些调度方式来控制 Pod 的调度了，主要有两个概念：**亲和性和反亲和性**，亲和性又分成节点亲和性(nodeAffinity)和 Pod 亲和性(podAffinity)。

### nodeSelector

在了解亲和性之前，我们先来了解一个非常常用的调度方式：nodeSelector。我们知道 label 标签是 kubernetes 中一个非常重要的概念，用户可以非常灵活的利用 label 来管理集群中的资源，比如最常见的 Service 对象通过 label 去匹配 Pod 资源，而 Pod 的调度也可以根据节点的 label 来进行调度。

我们可以通过下面的命令查看我们的 node 的 label：



```shell
➜ kubectl get nodes --show-labels
NAME      STATUS   ROLES                  AGE   VERSION   LABELS
master1   Ready    control-plane,master   82d   v1.22.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master1,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node.kubernetes.io/exclude-from-external-load-balancers=
node1     Ready    <none>                 82d   v1.22.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node1,kubernetes.io/os=linux
node2     Ready    <none>                 82d   v1.22.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node2,kubernetes.io/os=linux
```

现在我们先给节点 node2 增加一个com=youdianzhishi的标签，命令如下：



```shell
➜ kubectl label nodes node2 com=youdianzhishi
node/node2 labeled
```

我们可以通过上面的 --show-labels 参数可以查看上述标签是否生效。当节点被打上了相关标签后，在调度的时候就可以使用这些标签了，只需要在 Pod 的 spec 字段中添加 nodeSelector 字段，里面是我们需要被调度的节点的 label 标签，比如，下面的 Pod 我们要强制调度到 node2 这个节点上去，我们就可以使用 nodeSelector 来表示了：



```yaml
# node-selector-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: busybox-pod
  name: test-busybox
spec:
  containers:
  - command:
    - sleep
    - "3600"
    image: busybox
    imagePullPolicy: Always
    name: test-busybox
  nodeSelector:
    com: youdianzhishi
```

然后我们可以通过 describe 命令查看调度结果：



```yaml
➜ kubectl apply -f pod-selector-demo.yaml
pod/test-busybox created
➜ kubectl describe pod test-busybox
Name:         test-busybox
Namespace:    default
Priority:     0
Node:         node2/192.168.31.46
......
Node-Selectors:  com=youdianzhishi
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From                 Message
  ----    ------     ----       ----                 -------
  Normal  Scheduled  <unknown>  default-scheduler    Successfully assigned default/test-busybox to node2
  Normal  Pulling    13s        kubelet, node2  Pulling image "busybox"
  Normal  Pulled     10s        kubelet, node2  Successfully pulled image "busybox"
  Normal  Created    10s        kubelet, node2  Created container test-busybox
  Normal  Started    9s         kubelet, node2  Started container test-busybox
```

我们可以看到 Events 下面的信息，我们的 Pod 通过默认的 default-scheduler 调度器被绑定到了 node2 节点。不过需要注意的是nodeSelector 属于强制性的，如果我们的目标节点没有可用的资源，我们的 Pod 就会一直处于 Pending 状态。

通过上面的例子我们可以感受到 nodeSelector 的方式比较直观，但是还够灵活，控制粒度偏大，接下来我们再和大家了解下更加灵活的方式：节点亲和性(nodeAffinity)。

### 亲和性和反亲和性调度

前面我们了解了 kubernetes 调度器的调度流程，我们知道默认的调度器在使用的时候，经过了 predicates 和 priorities 两个阶段，但是在实际的生产环境中，往往我们需要根据自己的一些实际需求来控制 Pod 的调度，这就需要用到 nodeAffinity(节点亲和性)、podAffinity(pod 亲和性) 以及 podAntiAffinity(pod 反亲和性)。

亲和性调度可以分成**软策略**和**硬策略**两种方式:

- 软策略就是如果现在没有满足调度要求的节点的话，Pod 就会忽略这条规则，继续完成调度过程，说白了就是满足条件最好了，没有的话也无所谓
- 硬策略就比较强硬了，如果没有满足条件的节点的话，就不断重试直到满足条件为止，简单说就是你必须满足我的要求，不然就不干了

对于亲和性和反亲和性都有这两种规则可以设置： preferredDuringSchedulingIgnoredDuringExecution 和requiredDuringSchedulingIgnoredDuringExecution，前面的就是软策略，后面的就是硬策略。

### 节点亲和性

节点亲和性（nodeAffinity）主要是用来控制 Pod 要部署在哪些节点上，以及不能部署在哪些节点上的，它可以进行一些简单的逻辑组合了，不只是简单的相等匹配。

比如现在我们用一个 Deployment 来管理8个 Pod 副本，现在我们来控制下这些 Pod 的调度，如下例子：



```yaml
# node-affinity-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-affinity
  labels:
    app: node-affinity
spec:
  replicas: 8
  selector:
    matchLabels:
      app: node-affinity
  template:
    metadata:
      labels:
        app: node-affinity
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
          name: nginxweb
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:  # 硬策略
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: NotIn
                values:
                - master1
          preferredDuringSchedulingIgnoredDuringExecution:  # 软策略
          - weight: 1
            preference:
              matchExpressions:
              - key: com
                operator: In
                values:
                - youdianzhishi
```

上面这个 Pod 首先是要求不能运行在 master1 这个节点上，如果有个节点满足 com=youdianzhishi 的话就优先调度到这个节点上。

由于上面 node02 节点我们打上了 com=youdianzhishi 这样的 label 标签，所以按要求会优先调度到这个节点来的，现在我们来创建这个 Pod，然后查看具体的调度情况是否满足我们的要求。



```shell
➜ kubectl apply -f node-affinty-demo.yaml
deployment.apps/node-affinity created
➜ kubectl get pods -l app=node-affinity -o wide
NAME                            READY   STATUS    RESTARTS   AGE     IP             NODE         NOMINATED NODE   READINESS GATES
node-affinity-cdd9d54d9-bgbbh   1/1     Running   0          2m28s   10.244.2.247   node2   <none>           <none>
node-affinity-cdd9d54d9-dlbck   1/1     Running   0          2m28s   10.244.4.16    node1   <none>           <none>
node-affinity-cdd9d54d9-g2jr6   1/1     Running   0          2m28s   10.244.4.17    node1   <none>           <none>
node-affinity-cdd9d54d9-gzr58   1/1     Running   0          2m28s   10.244.1.118   node1   <none>           <none>
node-affinity-cdd9d54d9-hcv7r   1/1     Running   0          2m28s   10.244.2.246   node2   <none>           <none>
node-affinity-cdd9d54d9-kvxw4   1/1     Running   0          2m28s   10.244.2.245   node2   <none>           <none>
node-affinity-cdd9d54d9-p4mmk   1/1     Running   0          2m28s   10.244.2.244   node2   <none>           <none>
node-affinity-cdd9d54d9-t5mff   1/1     Running   0          2m28s   10.244.1.117   node2   <none>           <none>
```

从结果可以看出有5个 Pod 被部署到了 node2 节点上，但是可以看到并没有一个 Pod 被部署到 master1 这个节点上，因为我们的硬策略就是不允许部署到该节点上，而 node2 是软策略，所以会尽量满足。这里的匹配逻辑是 label 标签的值在某个列表中，现在 Kubernetes 提供的操作符有下面的几种：

- In：label 的值在某个列表中
- NotIn：label 的值不在某个列表中
- Gt：label 的值大于某个值
- Lt：label 的值小于某个值
- Exists：某个 label 存在
- DoesNotExist：某个 label 不存在

但是需要注意的是如果 nodeSelectorTerms 下面有多个选项的话，满足任何一个条件就可以了；如果 matchExpressions有多个选项的话，则必须同时满足这些条件才能正常调度 Pod。

### Pod 亲和性

Pod 亲和性（podAffinity）主要解决 Pod 可以和哪些 Pod 部署在同一个拓扑域中的问题（其中拓扑域用主机标签实现，可以是单个主机，也可以是多个主机组成的 cluster、zone 等等），而 Pod 反亲和性主要是解决 Pod 不能和哪些 Pod 部署在同一个拓扑域中的问题，它们都是处理的 Pod 与 Pod 之间的关系，比如一个 Pod 在一个节点上了，那么我这个也得在这个节点，或者你这个 Pod 在节点上了，那么我就不想和你待在同一个节点上。

由于我们这里只有一个集群，并没有区域或者机房的概念，所以我们这里直接使用主机名来作为拓扑域，把 Pod 创建在同一个主机上面。



```shell
➜ kubectl get nodes --show-labels
NAME      STATUS   ROLES                  AGE   VERSION   LABELS
master1   Ready    control-plane,master   82d   v1.22.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master1,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node.kubernetes.io/exclude-from-external-load-balancers=
node1     Ready    <none>                 82d   v1.22.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node1,kubernetes.io/os=linux
node2     Ready    <none>                 82d   v1.22.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node2,kubernetes.io/os=linux,com=youdianzhishi
```

同样，还是针对上面的资源对象，我们来测试下 Pod 的亲和性：



```yaml
# pod-affinity-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-affinity
  labels:
    app: pod-affinity
spec:
  replicas: 3
  selector:
    matchLabels:
      app: pod-affinity
  template:
    metadata:
      labels:
        app: pod-affinity
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: nginxweb
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:  # 硬策略
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - busybox-pod
            topologyKey: kubernetes.io/hostname
```

上面这个例子中的 Pod 需要调度到某个指定的节点上，并且该节点上运行了一个带有 app=busybox-pod 标签的 Pod。我们可以查看有标签 app=busybox-pod 的 pod 列表：



```shell
➜ kubectl get pods -l app=busybox-pod -o wide
NAME           READY   STATUS    RESTARTS   AGE   IP             NODE         NOMINATED NODE   READINESS GATES
test-busybox   1/1     Running   0          27m   10.244.2.242   node2   <none>           <none>
```

我们看到这个 Pod 运行在了 node2 的节点上面，所以按照上面的亲和性来说，上面我们部署的3个 Pod 副本也应该运行在 node2 节点上：



```shell
➜ kubectl apply -f pod-affinity-demo.yaml
deployment.apps/pod-affinity created
➜ kubectl get pods -o wide -l app=pod-affinity
NAME                            READY   STATUS    RESTARTS   AGE   IP             NODE         NOMINATED NODE   READINESS GATES
pod-affinity-587f9b5b58-5nxmf   1/1     Running   0          26s   10.244.2.249   node2   <none>           <none>
pod-affinity-587f9b5b58-m2j7s   1/1     Running   0          26s   10.244.2.248   node2   <none>           <none>
pod-affinity-587f9b5b58-vrd7b   1/1     Running   0          26s   10.244.2.250   node2   <none>           <none>
```

如果我们把上面的 test-busybox 和 pod-affinity 这个 Deployment 都删除，然后重新创建 pod-affinity 这个资源，看看能不能正常调度呢：



```shell
➜ kubectl delete -f node-selector-demo.yaml
pod "test-busybox" deleted
➜  kubectl delete -f pod-affinity-demo.yaml
deployment.apps "pod-affinity" deleted
➜ kubectl apply -f pod-affinity-demo.yaml
deployment.apps/pod-affinity created
➜ kubectl get pods -o wide -l app=pod-affinity
NAME                            READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
pod-affinity-587f9b5b58-bbfgr   0/1     Pending   0          18s   <none>   <none>   <none>           <none>
pod-affinity-587f9b5b58-lwc8n   0/1     Pending   0          18s   <none>   <none>   <none>           <none>
pod-affinity-587f9b5b58-pc7ql   0/1     Pending   0          18s   <none>   <none>   <none>           <none>
```

我们可以看到都处于 Pending 状态了，这是因为现在没有一个节点上面拥有 app=busybox-pod 这个标签的 Pod，而上面我们的调度使用的是硬策略，所以就没办法进行调度了，大家可以去尝试下重新将 test-busybox 这个 Pod 调度到其他节点上，观察下上面的3个副本会不会也被调度到对应的节点上去。

我们这个地方使用的是 kubernetes.io/hostname 这个**拓扑域**，意思就是我们当前调度的 Pod 要和目标的 Pod 处于同一个主机上面，因为要处于同一个拓扑域下面，为了说明这个问题，我们把拓扑域改成 beta.kubernetes.io/os，同样的我们当前调度的 Pod 要和目标的 Pod 处于同一个拓扑域中，目标的 Pod 是拥有 beta.kubernetes.io/os=linux 的标签，而我们这里所有节点都有这样的标签，这也就意味着我们所有节点都在同一个拓扑域中，所以我们这里的 Pod 可以被调度到任何一个节点，重新运行上面的 app=busybox-pod 的 Pod，然后再更新下我们这里的资源对象：



```shell
➜ kubectl get pods -o wide -l app=pod-affinity
NAME                           READY   STATUS    RESTARTS   AGE     IP             NODE         NOMINATED NODE   READINESS GATES
pod-affinity-76c56567c-792n4   1/1     Running   0          2m59s   10.244.2.254   node2   <none>           <none>
pod-affinity-76c56567c-8s2pd   1/1     Running   0          3m53s   10.244.4.18    node1   <none>           <none>
pod-affinity-76c56567c-hx7ck   1/1     Running   0          2m52s   10.244.3.23    node2   <none>           <none>
```

可以看到现在是分别运行在2个节点下面的，因为他们都属于 beta.kubernetes.io/os 这个拓扑域。

### Pod 反亲和性

Pod 反亲和性（podAntiAffinity）则是反着来的，比如一个节点上运行了某个 Pod，那么我们的模板 Pod 则不希望被调度到这个节点上面去了。我们把上面的 podAffinity 直接改成 podAntiAffinity：



```yaml
# pod-antiaffinity-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-antiaffinity
  labels:
    app: pod-antiaffinity
spec:
  replicas: 3
  selector:
    matchLabels:
      app: pod-antiaffinity
  template:
    metadata:
      labels:
        app: pod-antiaffinity
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: nginxweb
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:  # 硬策略
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - busybox-pod
            topologyKey: kubernetes.io/hostname
```

这里的意思就是如果一个节点上面有一个 app=busybox-pod 这样的 Pod 的话，那么我们的 Pod 就别调度到这个节点上面来，上面我们把app=busybox-pod 这个 Pod 固定到了 node2 这个节点上面的，所以正常来说我们这里的 Pod 不会出现在该节点上：



```shell
➜ kubectl apply -f pod-antiaffinity-demo.yaml
deployment.apps/pod-antiaffinity created
➜ kubectl get pods -l app=pod-antiaffinity -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
pod-antiaffinity-84d5bf9df4-9c9qk   1/1     Running   0          73s   10.244.4.19   node1   <none>           <none>
pod-antiaffinity-84d5bf9df4-q6lkm   1/1     Running   0          67s   10.244.3.24   node1   <none>           <none>
pod-antiaffinity-84d5bf9df4-vk9tc   1/1     Running   0          57s   10.244.3.25   node1   <none>           <none>
```

我们可以看到没有被调度到 node2 节点上，因为我们这里使用的是 Pod 反亲和性。大家可以思考下，如果这里我们将拓扑域更改成 beta.kubernetes.io/os 会怎么样呢？可以自己去测试下看看。

### 污点与容忍

对于 nodeAffinity 无论是硬策略还是软策略方式，都是调度 Pod 到预期节点上，而污点（Taints）恰好与之相反，如果一个节点标记为 Taints ，除非 Pod 也被标识为可以容忍污点节点，否则该 Taints 节点不会被调度 Pod。

比如用户希望把 Master 节点保留给 Kubernetes 系统组件使用，或者把一组具有特殊资源预留给某些 Pod，则污点就很有用了，Pod 不会再被调度到 taint 标记过的节点。我们使用 kubeadm 搭建的集群默认就给 master 节点添加了一个污点标记，所以我们看到我们平时的 Pod 都没有被调度到 master 上去：



```shell
➜ kubectl describe node master1
Name:               master1
Roles:              master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=master1
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/master=
......
Taints:             node-role.kubernetes.io/master:NoSchedule
Unschedulable:      false
......
```

我们可以使用上面的命令查看 master 节点的信息，其中有一条关于 Taints 的信息：node-role.kubernetes.io/master:NoSchedule，就表示master 节点打了一个污点的标记，其中影响的参数是 NoSchedule，表示 Pod 不会被调度到标记为 taints 的节点，除了 NoSchedule 外，还有另外两个选项：

- PreferNoSchedule：NoSchedule 的软策略版本，表示尽量不调度到污点节点上去
- NoExecute：该选项意味着一旦 Taint 生效，如该节点内正在运行的 Pod 没有对应容忍（Tolerate）设置，则会直接被逐出

污点 taint 标记节点的命令如下：



```shell
➜ kubectl taint nodes node2 test=node2:NoSchedule
node "node2" tainted
```

上面的命名将 node2 节点标记为了污点，影响策略是 NoSchedule，只会影响新的 Pod 调度，如果仍然希望某个 Pod 调度到 taint 节点上，则必须在 Spec 中做出 Toleration 定义，才能调度到该节点，比如现在我们想要将一个 Pod 调度到 master 节点：



```yaml
# taint-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: taint
  labels:
    app: taint
spec:
  replicas: 3
  selector:
    matchLabels:
      app: taint
  template:
    metadata:
      labels:
        app: taint
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - name: http
          containerPort: 80
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
```

由于 master 节点被标记为了污点，所以我们这里要想 Pod 能够调度到改节点去，就需要增加容忍的声明：



```yaml
tolerations:
- key: "node-role.kubernetes.io/master"
  operator: "Exists"
  effect: "NoSchedule"
```

然后创建上面的资源，查看结果：



```shell
➜ kubectl apply -f taint-demo.yaml
deployment.apps "taint" created
➜ kubectl get pods -o wide
NAME                                      READY     STATUS             RESTARTS   AGE       IP             NODE
......
taint-845d8bb4fb-57mhm                    1/1       Running            0          1m        10.244.4.247   node2
taint-845d8bb4fb-bbvmp                    1/1       Running            0          1m        10.244.0.33    master1
taint-845d8bb4fb-zb78x                    1/1       Running            0          1m        10.244.4.246   node2
......
```

我们可以看到有一个 Pod 副本被调度到了 master 节点，这就是容忍的使用方法。

对于 tolerations 属性的写法，其中的 key、value、effect 与 Node 的 Taint 设置需保持一致， 还有以下几点说明：

- 如果 operator 的值是 Exists，则 value 属性可省略
- 如果 operator 的值是 Equal，则表示其 key 与 value 之间的关系是 equal(等于)
- 如果不指定 operator 属性，则默认值为 Equal

另外，还有两个特殊值：

- 空的 key 如果再配合 Exists 就能匹配所有的 key 与 value，也就是是能容忍所有节点的所有 Taints
- 空的 effect 匹配所有的 effect

最后如果我们要取消节点的污点标记，可以使用下面的命令：



```shell
➜ kubectl taint nodes node2 test-
node "node2" untainted
```

### 课后习题

**1.不用 DaemonSet，如何使用 Deployment 是否实现同样的功能？**

我们知道 DaemonSet 控制器的功能就是在每个节点上运行一个 Pod，如何要使用 Deployment 来实现，首先就要设置副本数量为节点数，比如我们这里加上 master 节点一共3个节点，则要设置3个副本，要在 master 节点上执行自然要添加容忍，那么要如何保证一个节点上只运行一个 Pod 呢？是不是前面的提到的 Pod 反亲和性就可以实现，以自己 Pod 的标签来进行过滤校验即可，新的 Pod 不能运行在一个已经具有该 Pod 的节点上，是不是就是一个节点只能运行一个？模拟的资源清单如下所示：



```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mock-ds-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mock-ds-demo
  template:
    metadata:
      labels:
        app: mock-ds-demo
    spec:
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
          name: ngpt
      affinity:
        podAntiAffinity:  # pod反亲合性
          requiredDuringSchedulingIgnoredDuringExecution:  # 硬策略
          - labelSelector:
              matchExpressions:
              - key: app   # Pod的标签
                operator: In
                values: ["mock-ds-demo"]
            topologyKey: kubernetes.io/hostname  # 以hostname为拓扑域
```

创建上面的资源清单验证：



```plain
➜ kubectl get nodes
NAME      STATUS   ROLES                  AGE   VERSION
master1   Ready    control-plane,master   84d   v1.22.2
node1     Ready    <none>                 84d   v1.22.2
node2     Ready    <none>                 84d   v1.22.2
➜ kubectl get pods -l app=mock-ds-demo -o wide
NAME                            READY   STATUS    RESTARTS   AGE   IP             NODE      NOMINATED NODE   READINESS GATES
mock-ds-demo-8694759c69-tgqld   1/1     Running   0          30s   10.244.1.198   node1     <none>           <none>
mock-ds-demo-8694759c69-wtnwv   1/1     Running   0          30s   10.244.2.29    node2     <none>           <none>
mock-ds-demo-8694759c69-zt9pp   1/1     Running   0          30s   10.244.0.135   master1   <none>           <none>
```

可以看到我们用 Deployment 部署的服务在每个节点上都运行了一个 Pod，实现的效果和 DaemonSet 是一致的。

**2.同样的如果想在每个节点（或指定的一些节点）上运行2个（或多个）Pod 副本，如何实现？**

DaemonSet 是在每个节点上运行1个 Pod 副本，显然我们去创建2个（或多个）DaemonSet 即可实现该目标，但是这不是一个好的接近方案，而 PodAntiAffinity 只能将一个 Pod 调度到某个拓扑域中去，所以都不能很好的来解决这个问题。

要实现这种更细粒度的控制，我们可以通过设置拓扑分布约束来进行调度，设置拓扑分布约束来将 Pod 分布到不同的拓扑域下，从而实现高可用性或节省成本，具体实现方式请看下文。



## Pod 拓扑分布约束

在 k8s 集群调度中，**亲和性**相关的概念本质上都是控制 Pod 如何被调度 -- **堆叠或打散**。podAffinity 以及 podAntiAffinity 两个特性对 Pod 在不同拓扑域的分布进行了一些控制，podAffinity 可以将无数个 Pod 调度到特定的某一个拓扑域，这是**堆叠**的体现；podAntiAffinity 则可以控制一个拓扑域只存在一个 Pod，这是**打散**的体现。但这两种情况都太极端了，在不少场景下都无法达到理想的效果，例如为了实现容灾和高可用，将业务 Pod 尽可能均匀的分布在不同可用区就很难实现。

PodTopologySpread（Pod 拓扑分布约束） 特性的提出正是为了对 Pod 的调度分布提供更精细的控制，以提高服务可用性以及资源利用率，PodTopologySpread 由 EvenPodsSpread 特性门所控制，在 v1.16 版本第一次发布，并在 v1.18 版本进入 beta 阶段默认启用。

### 使用规范

在 Pod 的 Spec 规范中新增了一个 topologySpreadConstraints 字段即可配置拓扑分布约束，如下所示：



```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: <integer>
      topologyKey: <string>
      whenUnsatisfiable: <string>
      labelSelector: <object>
```

由于这个新增的字段是在 Pod spec 层面添加，因此更高层级的控制 (Deployment、DaemonSet、StatefulSet) 也能使用 PodTopologySpread 功能。

![img](https://cdn.nlark.com/yuque/0/2024/png/12365259/1705820580375-db3bf0a3-6594-4b37-8ed6-0bcfb4854253.png)

让我们结合上图来理解 topologySpreadConstraints 中各个字段的含义和作用：

- labelSelector: 用来查找匹配的 Pod，我们能够计算出每个拓扑域中匹配该 label selector 的 Pod 数量，在上图中，假如 label selector 是 app:foo，那么 zone1 的匹配个数为 2， zone2 的匹配个数为 0。
- topologyKey: 是 Node label 的 key，如果两个 Node 的 label 同时具有该 key 并且值相同，就说它们在同一个拓扑域。在上图中，指定 topologyKey 为 zone， 则具有 zone=zone1 标签的 Node 被分在一个拓扑域，具有 zone=zone2 标签的 Node 被分在另一个拓扑域。
- maxSkew: 这个属性理解起来不是很直接，它描述了 Pod **在不同拓扑域中不均匀分布的最大程度**（指定拓扑类型中任意两个拓扑域中匹配的 Pod 之间的最大允许差值），它必须大于零。每个拓扑域都有一个 skew 值，计算的公式是: skew[i] = 拓扑域[i]中匹配的 Pod 个数 - min{其他拓扑域中匹配的 Pod 个数}。在上图中，我们新建一个带有 app=foo 标签的 Pod：
- 如果该 Pod 被调度到 zone1，那么 zone1 中 Node 的 skew 值变为 3，zone2 中 Node 的 skew 值变为 0 (zone1 有 3 个匹配的 Pod，zone2 有 0 个匹配的 Pod )
- 如果该 Pod 被调度到 zone2，那么 zone1 中 Node 的 skew 值变为 2，zone2 中 Node 的 skew 值变为 1(zone2 有 1 个匹配的 Pod，拥有全局最小匹配 Pod 数的拓扑域正是 zone2 自己 )，则它满足maxSkew: 1 的约束（差值为 1）
- whenUnsatisfiable: 描述了如果 Pod 不满足分布约束条件该采取何种策略：
- **DoNotSchedule** (默认) 告诉调度器不要调度该 Pod，因此也可以叫作硬策略；
- **ScheduleAnyway** 告诉调度器根据每个 Node 的 skew 值打分排序后仍然调度，因此也可以叫作软策略。

### 单个拓扑约束

假设你拥有一个 4 节点集群，其中标记为  foo:bar  的 3 个 Pod 分别位于 node1、node2 和 node3 中：

![img](https://cdn.nlark.com/yuque/0/2024/png/12365259/1705820581250-ac1b0979-70da-41d1-8a9f-080aa89028d2.png)

如果希望新来的 Pod 均匀分布在现有的可用区域，则可以按如下设置其约束：



```yaml
kind: Pod
apiVersion: v1
metadata:
  name: mypod
  labels:
    foo: bar
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          foo: bar
  containers:
    - name: pause
      image: k8s.gcr.io/pause:3.1
```

topologyKey: zone  意味着均匀分布将只应用于存在标签键值对为 zone:<any value> 的节点。 whenUnsatisfiable: DoNotSchedule  告诉调度器如果新的 Pod 不满足约束，则不可调度。如果调度器将新的 Pod 放入 "zoneA"，Pods 分布将变为 [3, 1]，因此实际的偏差为 2(3 - 1)，这违反了  maxSkew: 1  的约定。此示例中，新 Pod 只能放置在 "zoneB" 上：

![img](https://cdn.nlark.com/yuque/0/2024/png/12365259/1705820580346-6ae39084-f2f4-479b-bb19-66af686bc654.png)

或者

![img](https://cdn.nlark.com/yuque/0/2024/png/12365259/1705820580365-9ae32085-031d-4522-b637-cf95991dc6a2.png)

你可以调整 Pod 约束以满足各种要求：

- 将  maxSkew  更改为更大的值，比如 "2"，这样新的 Pod 也可以放在 "zoneA" 上。
- 将  topologyKey  更改为 "node"，以便将 Pod 均匀分布在节点上而不是区域中。 在上面的例子中，如果  maxSkew  保持为 "1"，那么传入的 Pod 只能放在 "node4" 上。
- 将  whenUnsatisfiable: DoNotSchedule  更改为  whenUnsatisfiable: ScheduleAnyway， 以确保新的 Pod 可以被调度。

### 多个拓扑约束

上面是单个 Pod 拓扑分布约束的情况，下面的例子建立在前面例子的基础上来对多个 Pod 拓扑分布约束进行说明。假设你拥有一个 4 节点集群，其中 3 个标记为  foo:bar  的 Pod 分别位于 node1、node2 和 node3 上：

![img](https://cdn.nlark.com/yuque/0/2024/png/12365259/1705820580441-2f19a5b8-865c-4373-864d-440fec6396c5.png)

我们可以使用 2 个 TopologySpreadConstraint 来控制 Pod 在区域和节点两个维度上的分布：



```yaml
# two-constraints.yaml
kind: Pod
apiVersion: v1
metadata:
  name: mypod
  labels:
    foo: bar
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          foo: bar
    - maxSkew: 1
      topologyKey: node
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          foo: bar
  containers:
    - name: pause
      image: k8s.gcr.io/pause:3.1
```

在这种情况下，为了匹配第一个约束，新的 Pod 只能放置在 "zoneB" 中；而在第二个约束中， 新的 Pod 只能放置在 "node4" 上，最后两个约束的结果加在一起，唯一可行的选择是放置 在 "node4" 上。

多个约束之间是可能存在冲突的，假设有一个跨越 2 个区域的 3 节点集群：

![img](https://cdn.nlark.com/yuque/0/2024/png/12365259/1705820581633-9b50208e-a05b-40c1-8523-e73606365e21.png)

如果对集群应用 two-constraints.yaml，会发现 "mypod" 处于  Pending  状态，这是因为为了满足第一个约束，"mypod" 只能放在 "zoneB" 中，而第二个约束要求 "mypod" 只能放在 "node2" 上，Pod 调度无法满足这两种约束，所以就冲突了。

为了克服这种情况，你可以增加  maxSkew  或修改其中一个约束，让其使用  whenUnsatisfiable: ScheduleAnyway。

### 与 NodeSelector/NodeAffinity 一起使用

仔细观察可能你会发现我们并没有类似于 topologyValues 的字段来限制 Pod 将被调度到哪些拓扑去，默认情况会搜索所有节点并按 topologyKey 对其进行分组。有时这可能不是理想的情况，比如假设有一个集群，其节点标记为 env=prod、env=staging和 env=qa，现在你想跨区域将 Pod 均匀地放置到 qa 环境中，是否可行?

答案是肯定的，我们可以结合 NodeSelector 或 NodeAffinity 一起使用，PodTopologySpread 会计算满足选择器的节点之间的传播约束。

![img](https://cdn.nlark.com/yuque/0/2024/png/12365259/1705820583878-f5a2909c-22e9-4ddd-ae21-cabee5c407f7.png)

如上图所示我们可以通过指定 spec.affinity.nodeAffinity 将**搜索范围**限制为 qa 环境，在该范围内 Pod 将被调度到一个满足 topologySpreadConstraints 的区域，这里就只能被调度到 zone=zone2 的节点上去了。

### 集群默认约束

除了为单个 Pod 设置拓扑分布约束，也可以为集群设置默认的拓扑分布约束，默认拓扑分布约束在且仅在以下条件满足 时才会应用到 Pod 上：

- Pod 没有在其  .spec.topologySpreadConstraints  设置任何约束；
- Pod 隶属于某个服务、副本控制器、ReplicaSet 或 StatefulSet。

你可以在  [调度方案（Schedulingg Profile）](https://kubernetes.io/zh/docs/reference/scheduling/config/#profiles)中将默认约束作为  PodTopologySpread  插件参数的一部分来进行设置。 约束的设置采用和前面 Pod 中的规范一致，只是  labelSelector  必须为空。配置的示例可能看起来像下面这个样子：



```yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration

profiles:
  - pluginConfig:
      - name: PodTopologySpread
        args:
          defaultConstraints:
            - maxSkew: 1
              topologyKey: topology.kubernetes.io/zone
              whenUnsatisfiable: ScheduleAnyway
          defaultingType: List
```

### 课后习题

现在我们再去解决上节课留下的一个问题 - **如果想在每个节点（或指定的一些节点）上运行 2 个（或多个）Pod 副本，如何实现？**

这里以我们的集群为例，加上 master 节点一共有 3 个节点，每个节点运行 2 个副本，总共就需要 6 个 Pod 副本，要在 master 节点上运行，则同样需要添加容忍，如果只想在一个节点上运行 2 个副本，则可以使用我们的拓扑分布约束来进行细粒度控制，对应的资源清单如下所示：



```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: topo-demo
spec:
  replicas: 6
  selector:
    matchLabels:
      app: topo
  template:
    metadata:
      labels:
        app: topo
    spec:
      tolerations:
        - key: "node-role.kubernetes.io/master"
          operator: "Exists"
          effect: "NoSchedule"
      containers:
        - image: nginx
          name: nginx
          ports:
            - containerPort: 80
              name: ngpt
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: topo
```

这里我们重点需要关注的就是 topologySpreadConstraints 部分的配置，我们选择使用 kubernetes.io/hostname 为拓扑域，相当于就是 3 个节点都是独立的，maxSkew: 1 表示最大的分布不均匀度为 1，所以只能出现的调度结果就是每个节点运行 2 个 Pod。

![img](https://cdn.nlark.com/yuque/0/2024/png/12365259/1705820582424-19c6e69a-4d35-43e7-9791-41271dcce381.png)

直接创建上面的资源即可验证：



```shell
➜ kubectl get nodes
NAME      STATUS   ROLES                  AGE   VERSION
master1   Ready    control-plane,master   85d   v1.22.2
node1     Ready    <none>                 85d   v1.22.2
node2     Ready    <none>                 85d   v1.22.2
➜ kubectl get pods -l app=topo -o wide
NAME                         READY   STATUS    RESTARTS   AGE     IP             NODE      NOMINATED NODE   READINESS GATES
topo-demo-6bbf65d967-7969w   1/1     Running   0          7m16s   10.244.2.40    node2     <none>           <none>
topo-demo-6bbf65d967-8vhb8   1/1     Running   0          7m16s   10.244.2.41    node2     <none>           <none>
topo-demo-6bbf65d967-cvg7j   1/1     Running   0          7m16s   10.244.1.211   node1     <none>           <none>
topo-demo-6bbf65d967-hzhv2   1/1     Running   0          7m16s   10.244.0.143   master1   <none>           <none>
topo-demo-6bbf65d967-nvg4z   1/1     Running   0          7m16s   10.244.0.144   master1   <none>           <none>
topo-demo-6bbf65d967-w7w29   1/1     Running   0          7m16s   10.244.1.212   node1     <none>           <none>
```