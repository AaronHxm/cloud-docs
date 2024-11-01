![img](https://cdn.nlark.com/yuque/0/2021/png/12365259/1639577561676-9a44a460-e95c-4628-aed4-9083da2349d4.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/12365259/1639577591165-b02b105b-0703-414b-8d78-1c6a6f638fde.png)

Kubernetes 目前支持多达 28 种数据卷类型（其中大部分特定于具体的云环境如 GCE/AWS/Azure 等），如需查阅所有的数据卷类型，请查阅 Kubernetes 官方文档 [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/) 。如：

- 非持久性存储 

- - emptyDir
  - HostPath

- 网络连接性存储

- - SAN：iSCSI、ScaleIO Volumes、FC (Fibre Channel)
  - NFS：nfs，cfs

- 分布式存储

- - Glusterfs
  - RBD (Ceph Block Device)
  - CephFS
  - Portworx Volumes
  - Quobyte Volumes

- 云端存储

- - GCEPersistentDisk
  - AWSElasticBlockStore
  - AzureFile
  - AzureDisk
  - Cinder (OpenStack block storage)
  - VsphereVolume
  - StorageOS

- 自定义存储

- - FlexVolume

# 配置

配置最佳实战: 

- 云原生 应用12要素 中，提出了配置分离。https://www.kdocs.cn/view/l/skIUQnbIc6cJ
- 在推送到集群之前，配置文件应存储在**版本控制**中。 这允许您在必要时快速回滚配置更改。 它还有助于集群重新创建和恢复。
- **使用 YAML 而不是 JSON 编写配置文件**。虽然这些格式几乎可以在所有场景中互换使用，但 YAML 往往更加用户友好。
- 建议相关对象分组到一个文件。比如 [guestbook-all-in-one.yaml](https://github.com/kubernetes/examples/tree/master/guestbook/all-in-one/guestbook-all-in-one.yaml)
- 除非必要，否则不指定默认值：简单的最小配置会降低错误的可能性。
- 将对象描述放在注释中，以便更好地进行内省。

## Secret

### 概念

说 ConfigMap 这个资源对象是 Kubernetes 当中非常重要的一个资源对象，一般情况下 ConfigMap 是用来存储一些非安全的配置信息，如果涉及到一些安全相关的数据的话用 ConfigMap 就非常不妥了，因为 ConfigMap 是明文存储的，这个时候我们就需要用到另外一个资源对象了：Secret，Secret用来保存敏感信息，例如密码、OAuth 令牌和 ssh key 等等，将这些信息放在 Secret 中比放在 Pod 的定义中或者 Docker 镜像中要更加安全和灵活。

Secret 主要使用的有以下三种类型：

- Opaque：base64 编码格式的 Secret，用来存储密码、密钥等；但数据也可以通过 base64 –decode 解码得到原始数据，所有加密性很弱。
- kubernetes.io/dockercfg: ~/.dockercfg 文件的序列化形式
- kubernetes.io/dockerconfigjson：用来存储私有docker registry的认证信息，~/.docker/config.json 文件的序列化形式
- kubernetes.io/service-account-token：用于 ServiceAccount, ServiceAccount 创建时 Kubernetes 会默认创建一个对应的 Secret 对象，Pod 如果使用了 ServiceAccount，对应的 Secret 会自动挂载到 Pod 目录 /run/secrets/kubernetes.io/serviceaccount 中
- kubernetes.io/ssh-auth：用于 SSH 身份认证的凭据
- kubernetes.io/basic-auth：用于基本身份认证的凭据
- bootstrap.kubernetes.io/token：用于节点接入集群的校验的 Secret

#### Opaque Secret

Secret 资源包含2个键值对： data 和 stringData，data 字段用来存储 base64 编码的任意数据，提供 stringData 字段是为了方便，它允许 Secret 使用未编码的字符串。 data 和 stringData 的键必须由字母、数字、-，_ 或 . 组成。

比如我们来创建一个用户名为 admin，密码为 admin321 的 Secret 对象，首先我们需要先把用户名和密码做 base64 编码：

```shell
➜  ~ echo -n "admin" | base64
YWRtaW4=
➜  ~ echo -n "admin321" | base64
YWRtaW4zMjE=
```

然后我们就可以利用上面编码过后的数据来编写一个 YAML 文件：(secret-demo.yaml)



```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: YWRtaW4zMjE=
```

然后我们就可以使用 kubectl 命令来创建了：

```yaml
➜  ~ kubectl apply -f secret-demo.yaml
secret "mysecret" created
```

利用get secret命令查看：

```shell
➜  ~ kubectl get secret
NAME                  TYPE                                  DATA      AGE
default-token-n9w2d   kubernetes.io/service-account-token   3         33d
mysecret              Opaque                                2         40s
```

其中 default-token-n9w2d 为创建集群时默认创建的 Secret，被 serviceacount/default 引用。我们可以使用 describe 命令查看详情：

```shell
➜  ~ kubectl describe secret mysecret
Name:         mysecret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  8 bytes
username:  5 bytes
```

我们可以看到利用 describe 命令查看到的 Data 没有直接显示出来，如果想看到 Data 里面的详细信息，同样我们可以输出成YAML 文件进行查看：



```yaml
➜  ~ kubectl get secret mysecret -o yaml
apiVersion: v1
data:
  password: YWRtaW4zMjE=
  username: YWRtaW4=
kind: Secret
metadata:
  creationTimestamp: 2018-06-19T15:27:06Z
  name: mysecret
  namespace: default
  resourceVersion: "3694084"
  selfLink: /api/v1/namespaces/default/secrets/mysecret
  uid: 39c139f5-73d5-11e8-a101-525400db4df7
type: Opaque
```

对于某些场景，你可能希望使用 stringData 字段，这字段可以将一个非 base64 编码的字符串直接放入 Secret 中， 当创建或更新该 Secret 时，此字段将被编码。

比如当我们部署应用时，使用 Secret 存储配置文件， 你希望在部署过程中，填入部分内容到该配置文件。例如，如果你的应用程序使用以下配置文件:

```yaml
apiUrl: "https://my.api.com/api/v1"
username: "<user>"
password: "<password>"
```

那么我们就可以使用以下定义将其存储在 Secret 中:



```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
stringData:
  config.yaml: |
    apiUrl: "https://my.api.com/api/v1"
    username: <user>
    password: <password>
```

比如我们直接创建上面的对象后重新获取对象的话 config.yaml 的值会被编码：



```yaml
➜  ~ kubectl get secret mysecret -o yaml
apiVersion: v1
data:
  config.yaml: YXBpVXJsOiAiaHR0cHM6Ly9teS5hcGkuY29tL2FwaS92MSIKdXNlcm5hbWU6IDx1c2VyPgpwYXNzd29yZDogPHBhc3N3b3JkPgo=
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Secret","metadata":{"annotations":{},"name":"mysecret","namespace":"default"},"stringData":{"config.yaml":"apiUrl: \"https://my.api.com/api/v1\"\nusername: \u003cuser\u003e\npassword: \u003cpassword\u003e\n"},"type":"Opaque"}
  creationTimestamp: "2021-11-21T10:42:25Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:config.yaml: {}
      f:metadata:
        f:annotations:
          .: {}
          f:kubectl.kubernetes.io/last-applied-configuration: {}
      f:type: {}
    manager: kubectl
    operation: Update
    time: "2021-11-21T10:42:25Z"
  name: mysecret
  namespace: default
  resourceVersion: "857340"
  uid: 5a28d296-5f53-4e4c-92f3-c1d7c952ace2
type: Opaque
```

#### 环境变量

创建好 Secret对象后，有两种方式来使用它：

- 以环境变量的形式
- 以Volume的形式挂载

首先我们来测试下环境变量的方式，同样的，我们来使用一个简单的 busybox 镜像来测试下:(secret1-pod.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret1-pod
spec:
  containers:
  - name: secret1
    image: busybox
    command: [ "/bin/sh", "-c", "env" ]
    env:
    - name: USERNAME
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: username
    - name: PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: password
```

主要需要注意的是上面环境变量中定义的 secretKeyRef 字段，和我们前文的 configMapKeyRef 类似，一个是从 Secret 对象中获取，一个是从 ConfigMap 对象中获取，创建上面的 Pod：

```shell
➜  ~ kubectl create -f secret1-pod.yaml
pod "secret1-pod" created
```

然后我们查看Pod的日志输出：



```shell
➜  ~ kubectl logs secret1-pod
...
USERNAME=admin
PASSWORD=admin321
...
```

可以看到有 USERNAME 和 PASSWORD 两个环境变量输出出来。

#### Volume 挂载

同样的我们用一个 Pod 来验证下 Volume 挂载，创建一个 Pod 文件：(secret2-pod.yaml)



```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret2-pod
spec:
  containers:
  - name: secret2
    image: busybox
    command: ["/bin/sh", "-c", "ls /etc/secrets"]
    volumeMounts:
    - name: secrets
      mountPath: /etc/secrets
  volumes:
  - name: secrets
    secret:
     secretName: mysecret
```

创建 Pod，然后查看输出日志：



```shell
➜  ~ kubectl create -f secret-pod2.yaml
pod "secret2-pod" created
➜  ~ kubectl logs secret2-pod
password
username
```

可以看到 Secret 把两个 key 挂载成了两个对应的文件。当然如果想要挂载到指定的文件上面，是不是也可以使用上一节课的方法：在 secretName 下面添加 items 指定 key 和 path，这个大家可以参考ConfigMap 中的方法去测试下。

#### kubernetes.io/dockerconfigjson


除了上面的 Opaque 这种类型外，我们还可以来创建用户 docker registry 认证的 Secret，直接使用`kubectl create 命令创建即可，如下：



```plain
➜  ~ kubectl create secret docker-registry myregistry --docker-server=DOCKER_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
secret "myregistry" created
```

除了上面这种方法之外，我们也可以通过指定文件的方式来创建镜像仓库认证信息，需要注意对应的 KEY 和 TYPE：

```shell
kubectl create secret generic myregistry --from-file=.dockerconfigjson=/root/.docker/config.json --type=kubernetes.io/dockerconfigjson
```

然后查看 Secret 列表：



```plain
➜  ~ kubectl get secret
NAME                  TYPE                                  DATA      AGE
default-token-n9w2d   kubernetes.io/service-account-token   3         33d
myregistry            kubernetes.io/dockerconfigjson        1         15s
mysecret              Opaque                                2         34m
```

注意看上面的 TYPE 类型，myregistry 对应的是 kubernetes.io/dockerconfigjson，同样的可以使用 describe 命令来查看详细信息：

```shell
➜  ~ kubectl describe secret myregistry
Name:         myregistry
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/dockerconfigjson

Data
====
.dockerconfigjson:  152 bytes
```

同样的可以看到 Data 区域没有直接展示出来，如果想查看的话可以使用 -o yaml 来输出展示出来：

```yaml
➜  ~ kubectl get secret myregistry -o yaml
apiVersion: v1
data:
  .dockerconfigjson: eyJhdXRocyI6eyJET0NLRVJfU0VSVkVSIjp7InVzZXJuYW1lIjoiRE9DS0VSX1VTRVIiLCJwYXNzd29yZCI6IkRPQ0tFUl9QQVNTV09SRCIsImVtYWlsIjoiRE9DS0VSX0VNQUlMIiwiYXV0aCI6IlJFOURTMFZTWDFWVFJWSTZSRTlEUzBWU1gxQkJVMU5YVDFKRSJ9fX0=
kind: Secret
metadata:
  creationTimestamp: 2018-06-19T16:01:05Z
  name: myregistry
  namespace: default
  resourceVersion: "3696966"
  selfLink: /api/v1/namespaces/default/secrets/myregistry
  uid: f91db707-73d9-11e8-a101-525400db4df7
type: kubernetes.io/dockerconfigjson
```

可以把上面的 data.dockerconfigjson 下面的数据做一个 base64 解码，看看里面的数据是怎样的呢？



```plain
➜  ~ echo eyJhdXRocyI6eyJET0NLRVJfU0VSVkVSIjp7InVzZXJuYW1lIjoiRE9DS0VSX1VTRVIiLCJwYXNzd29yZCI6IkRPQ0tFUl9QQVNTV09SRCIsImVtYWlsIjoiRE9DS0VSX0VNQUlMIiwiYXV0aCI6IlJFOURTMFZTWDFWVFJWSTZSRTlEUzBWU1gxQkJVMU5YVDFKRSJ9fX0= | base64 -d
{"auths":{"DOCKER_SERVER":{"username":"DOCKER_USER","password":"DOCKER_PASSWORD","email":"DOCKER_EMAIL","auth":"RE9DS0VSX1VTRVI6RE9DS0VSX1BBU1NXT1JE"}}}
```

如果我们需要拉取私有仓库中的 Docker 镜像的话就需要使用到上面的 myregistry 这个 Secret：



```plain
apiVersion: v1
kind: Pod
metadata:
  name: foo
spec:
  containers:
  - name: foo
    image: 192.168.1.100:5000/test:v1
  imagePullSecrets:
  - name: myregistry
```

**imagePullSecrets**

ImagePullSecrets 与 Secrets 不同，因为 Secrets 可以挂载到 Pod 中，但是 ImagePullSecrets 只能由 Kubelet 访问。

我们需要拉取私有仓库镜像 192.168.1.100:5000/test:v1，我们就需要针对该私有仓库来创建一个如上的 Secret，然后在 Pod 中指定 imagePullSecrets。

除了设置 Pod.spec.imagePullSecrets 这种方式来获取私有镜像之外，我们还可以通过在 ServiceAccount 中设置 imagePullSecrets，然后就会自动为使用该 SA 的 Pod 注入 imagePullSecrets 信息：



```plain
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2019-11-08T12:00:04Z"
  name: default
  namespace: default
  resourceVersion: "332"
  selfLink: /api/v1/namespaces/default/serviceaccounts/default
  uid: cc37a719-c4fe-4ebf-92da-e92c3e24d5d0
secrets:
- name: default-token-5tsh4
imagePullSecrets:
- name: myregistry
```

#### kubernetes.io/basic-auth

该类型用来存放用于基本身份认证所需的凭据信息，使用这种 Secret 类型时，Secret 的 data 字段（不一定）必须包含以下两个键（相当于是约定俗成的一个规定）：

- username: 用于身份认证的用户名
- password: 用于身份认证的密码或令牌

以上两个键的键值都是 base64 编码的字符串。 然你也可以在创建 Secret 时使用 stringData 字段来提供明文形式的内容。下面的 YAML 是基本身份认证 Secret 的一个示例清单：



```plain
apiVersion: v1
kind: Secret
metadata:
  name: secret-basic-auth
type: kubernetes.io/basic-auth
stringData:
  username: admin
  password: admin321
```

提供基本身份认证类型的 Secret 仅仅是出于用户方便性考虑，我们也可以使用 Opaque 类型来保存用于基本身份认证的凭据，不过使用内置的 Secret 类型的有助于对凭据格式进行统一处理。

#### kubernetes.io/ssh-auth

该类型用来存放 SSH 身份认证中所需要的凭据，使用这种 Secret 类型时，你就不一定必须在其 data（或 stringData）字段中提供一个 ssh-privatekey 键值对，作为要使用的 SSH 凭据。

如下所示是一个 SSH 身份认证 Secret 的配置示例：



```plain
apiVersion: v1
kind: Secret
metadata:
  name: secret-ssh-auth
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: |
          MIIEpQIBAAKCAQEAulqb/Y ...
```

同样提供 SSH 身份认证类型的 Secret 也仅仅是出于用户方便性考虑，我们也可以使用 Opaque 类型来保存用于 SSH 身份认证的凭据，只是使用内置的 Secret 类型的有助于对凭据格式进行统一处理。

#### kubernetes.io/tls

该类型用来存放证书及其相关密钥（通常用在 TLS 场合）。此类数据主要提供给 Ingress 资源，用以校验 TLS 链接，当使用此类型的 Secret 时，Secret 配置中的 data （或 stringData）字段必须包含 tls.key 和 tls.crt主键。下面的 YAML 包含一个 TLS Secret 的配置示例：



```plain
apiVersion: v1
kind: Secret
metadata:
  name: secret-tls
type: kubernetes.io/tls
data:
  tls.crt: |
        MIIC2DCCAcCgAwIBAgIBATANBgkqh ...
  tls.key: |
        MIIEpgIBAAKCAQEA7yn3bRHQ5FHMQ ...
```

提供 TLS 类型的 Secret 仅仅是出于用户方便性考虑，我们也可以使用 Opaque 类型来保存用于 TLS 服务器与/或客户端的凭据。不过，使用内置的 Secret 类型的有助于对凭据格式进行统一化处理。当使用 kubectl 来创建 TLS Secret 时，我们可以像下面的例子一样使用 tls 子命令：



```plain
➜  ~ kubectl create secret tls my-tls-secret \
  --cert=path/to/cert/file \
  --key=path/to/key/file
```

需要注意的是用于 --cert 的公钥证书必须是 .PEM 编码的 （Base64 编码的 DER 格式），且与 --key 所给定的私钥匹配，私钥必须是通常所说的 PEM 私钥格式，且未加密。对这两个文件而言，PEM 格式数据的第一行和最后一行（例如，证书所对应的 --------BEGIN CERTIFICATE----- 和 -------END CERTIFICATE----）都不会包含在其中。

#### kubernetes.io/service-account-token

另外一种 Secret 类型就是 kubernetes.io/service-account-token，用于被 ServiceAccount 引用。ServiceAccout 创建时 Kubernetes 会默认创建对应的 Secret，如下所示我们随意创建一个 Pod：



```plain
➜  ~ kubectl run secret-pod3 --image nginx:1.7.9
deployment.apps "secret-pod3" created
➜  ~ kubectl get pods
NAME                           READY     STATUS    RESTARTS   AGE
...
secret-pod3-78c8c76db8-7zmqm   1/1       Running   0          13s
...
```

我们可以直接查看这个 Pod 的详细信息：



```plain
volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-lvhfb
      readOnly: true
  ......
  serviceAccount: default
  serviceAccountName: default
  volumes:
  - name: kube-api-access-lvhfb
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
```

当创建 Pod 的时候，如果没有指定 ServiceAccount，Pod 则会使用命名空间中名为 default 的 ServiceAccount，上面我们可以看到 spec.serviceAccountName 字段已经被自动设置了。

可以看到这里通过一个 projected 类型的 Volume 挂载到了容器的 /var/run/secrets/kubernetes.io/serviceaccount 的目录中，projected 类型的 Volume 可以同时挂载多个来源的数据，这里我们挂载了一个 downwardAPI 来获取 namespace，通过 ConfigMap 来获取 ca.crt 证书，然后还有一个 serviceAccountToken 类型的数据源。

在之前的版本（v1.20）中，是直接将 default（自动创建的）的 ServiceAccount 对应的 Secret 对象通过 Volume 挂载到了容器的 /var/run/secrets/kubernetes.io/serviceaccount 的目录中的，现在的版本提供了更多的配置选项，比如上面我们配置了 expirationSeconds 和 path 两个属性。

前面我们也提到了默认情况下当前 namespace 下面的 Pod 会默认使用 default 这个 ServiceAccount，对应的 Secret 会自动挂载到 Pod 的 /var/run/secrets/kubernetes.io/serviceaccount/ 目录中，这样我们就可以在 Pod 里面获取到用于身份认证的信息了。

我们可以使用自动挂载给 Pod 的 ServiceAccount 凭据访问 API，我们也可以通过在 ServiceAccount 上设置 automountServiceAccountToken: false 来实现不给 ServiceAccount 自动挂载 API 凭据：



```plain
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
automountServiceAccountToken: false
...
```

此外也可以选择不给特定 Pod 自动挂载 API 凭据：



```plain
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: build-robot
  automountServiceAccountToken: false
  ...
```

如果 Pod 和 ServiceAccount 都指定了 automountServiceAccountToken 值，则 Pod 的 spec 优先于 ServiceAccount。

#### ServiceAccount Token 投影

ServiceAccount 是 Pod 和集群 apiserver 通讯的访问凭证，传统方式下，在 Pod 中使用 ServiceAccount 可能会遇到如下的安全挑战：

- ServiceAccount 中的 JSON Web Token (JWT) 没有绑定 audience 身份，因此所有 ServiceAccount 的使用者都可以彼此扮演，存在伪装攻击的可能
- 传统方式下每一个 ServiceAccount 都需要存储在一个对应的 Secret 中，并且会以文件形式存储在对应的应用节点上，而集群的系统组件在运行过程中也会使用到一些权限很高的 ServiceAccount，其增大了集群管控平面的攻击面，攻击者可以通过获取这些管控组件使用的 ServiceAccount 非法提权
- ServiceAccount 中的 JWT token 没有设置过期时间，当上述 ServiceAccount 泄露情况发生时，只能通过轮转 ServiceAccount 的签发私钥来进行防范
- 每一个 ServiceAccount 都需要创建一个与之对应的 Secret，在大规模的应用部署下存在弹性和容量风险

为解决这个问题 Kubernetes 提供了 ServiceAccount Token 投影特性用于增强 ServiceAccount 的安全性，ServiceAccount 令牌卷投影可使 Pod 支持以卷投影的形式将 ServiceAccount 挂载到容器中从而避免了对 Secret 的依赖。

通过 ServiceAccount 令牌卷投影可用于工作负载的 ServiceAccount 令牌是受时间限制，受 audience 约束的,并且不与 Secret 对象关联。如果删除了 Pod 或删除了 ServiceAccount，则这些令牌将无效，从而可以防止任何误用，Kubelet 还会在令牌即将到期时自动旋转令牌，另外，还可以配置希望此令牌可用的路径。

为了启用令牌请求投射（此功能在 Kubernetes 1.12 中引入，Kubernetes v1.20 已经稳定版本），你必须为 kube-apiserver 设置以下命令行参数，通过 kubeadm 安装的集群已经默认配置了：



```plain
--service-account-issuer  # serviceaccount token 中的签发身份，即token payload中的iss字段。
--service-account-key-file # token 私钥文件路径
--service-account-signing-key-file  # token 签名私钥文件路径
--api-audiences (可选参数)  # 合法的请求token身份，用于apiserver服务端认证请求token是否合法。
```

配置完成后就可以指定令牌的所需属性，例如身份和有效时间，这些属性在默认 ServiceAccount 令牌上无法配置。当删除 Pod 或 ServiceAccount 时，ServiceAccount 令牌也将对 API 无效。

我们可以使用名为 ServiceAccountToken 的 ProjectedVolume 类型在 PodSpec 上配置此功能，比如要向 Pod 提供具有 "vault" 用户以及两个小时有效期的令牌，可以在 PodSpec 中配置以下内容：

例如当 Pod 中需要使用 audience 为 vault 并且有效期为2个小时的 ServiceAccount 时，我们可以使用以下模板配置 PodSpec 来使用 ServiceAccount 令牌卷投影。



```plain
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot

---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    - mountPath: /var/run/secrets/tokens
      name: vault-token
  serviceAccountName: build-robot
  volumes:
  - name: vault-token
    projected:
      sources:
      - serviceAccountToken:
          path: vault-token
          expirationSeconds: 7200
          audience: vault
```

kubelet 组件会替 Pod 请求令牌并将其保存起来，通过将令牌存储到一个可配置的路径使之在 Pod 内可用，并在令牌快要到期的时候刷新它。 kubelet 会在令牌存在期达到其 TTL 的 80% 的时候或者令牌生命期超过 24 小时的时候主动轮换它。应用程序负责在令牌被轮换时重新加载其内容。对于大多数使用场景而言，周期性地（例如，每隔 5 分钟）重新加载就足够了。

#### 其他特性

如果某个容器已经在通过环境变量使用某 Secret，对该 Secret 的更新不会被容器马上看见，除非容器被重启，当然我们可以使用一些第三方的解决方案在 Secret 发生变化时触发容器重启。

在 Kubernetes v1.21 版本提供了不可变的 Secret 和 ConfigMap 的可选配置[stable]，我们可以设置 Secret 和 ConfigMap 为不可变的，对于大量使用 Secret 或者 ConfigMap 的集群（比如有成千上万各不相同的 Secret 供 Pod 挂载）时，禁止变更它们的数据有很多好处：

- 可以防止意外更新导致应用程序中断
- 通过将 Secret 标记为不可变来关闭 kube-apiserver 对其的 watch 操作，从而显著降低 kube-apiserver 的负载，提升集群性能

这个特性通过可以通过 ImmutableEmphemeralVolumes 特性门来进行开启，从 v1.19 开始默认启用，我们可以通过将 Secret 的 immutable 字段设置为 true 创建不可更改的 Secret。 例如：



```plain
apiVersion: v1
kind: Secret
metadata:
  ...
data:
  ...
immutable: true  # 标记为不可变
```

一旦一个 Secret 或 ConfigMap 被标记为不可更改，撤销此操作或者更改 data 字段的内容都是不允许的，只能删除并重新创建这个 Secret。现有的 Pod 将维持对已删除 Secret 的挂载点，所以我们也是建议重新创建这些 Pod。

#### Secret vs ConfigMap

最后我们来对比下 Secret 和 ConfigMap这两种资源对象的异同点：

##### 相同点

- key/value的形式
- 属于某个特定的命名空间
- 可以导出到环境变量
- 可以通过目录/文件形式挂载
- 通过 volume 挂载的配置信息均可热更新

##### 不同点

- Secret 可以被 ServerAccount 关联
- Secret 可以存储 docker register 的鉴权信息，用在 ImagePullSecret 参数中，用于拉取私有仓库的镜像
- Secret 支持 Base64 加密
- Secret 分为 kubernetes.io/service-account-token、kubernetes.io/dockerconfigjson、Opaque 三种类型，而 Configmap 不区分类型

**使用注意**

同样 Secret 文件大小限制为 1MB（ETCD 的要求）；Secret 虽然采用 Base64 编码，但是我们还是可以很方便解码获取到原始信息，所以对于非常重要的数据还是需要慎重考虑，可以考虑使用 [Vault](https://www.vaultproject.io/) 来进行加密管理。



### Secret实战

- Secret 对象类型用来**保存敏感信息**，例如密码、OAuth 令牌和 SSH 密钥。 将这些信息放在 secret 中比放在 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 的定义或者 [容器镜像](https://kubernetes.io/zh/docs/reference/glossary/?all=true#term-image) 中来说更加安全和灵活。
- Secret 是一种包含少量敏感信息例如密码、令牌或密钥的对象。用户可以创建 Secret，同时系统也创建了一些 Secret。

### Secret种类

![img](https://cdn.nlark.com/yuque/0/2021/png/12365259/1639577741128-78ac5e7c-686c-4427-b9c8-17e23fc7e449.png)

### 细分类型

![img](https://cdn.nlark.com/yuque/0/2021/png/12365259/1639577779858-9453a2de-b839-4df7-9d6c-664af564ccb6.png)

jenkin-----》 全局的凭证。 etcd（CP）===redis

### Pod如何引用

要使用 Secret，Pod 需要引用 Secret。 Pod 可以用三种方式之一来使用 Secret：

- 作为挂载到一个或多个容器上的 [卷](https://kubernetes.io/zh/docs/concepts/storage/volumes/) 中的[文件](https://kubernetes.io/zh/docs/concepts/configuration/secret/#using-secrets-as-files-from-a-pod)。（volume进行挂载）
- 作为[容器的环境变量](https://kubernetes.io/zh/docs/concepts/configuration/secret/#using-secrets-as-environment-variables)（envFrom字段引用）
- 由 [kubelet 在为 Pod 拉取镜像时使用](https://kubernetes.io/zh/docs/concepts/configuration/secret/#using-imagepullsecrets)（此时Secret是docker-registry类型的）

Secret 对象的名称必须是合法的 [DNS 子域名](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/names#dns-subdomain-names)。 在为创建 Secret 编写配置文件时，你可以设置 data 与/或 stringData 字段。 data 和 stringData 字段都是可选的。data 字段中所有键值都必须是 base64 编码的字符串。如果不希望执行这种 base64 字符串的转换操作，你可以选择设置 stringData 字段，其中可以使用任何字符串作为其取值。



### 实验

##### generic 类型

```yaml
## 命令行
#### 1、使用基本字符串
kubectl create secret generic dev-db-secret \
  --from-literal=username=devuser \
  --from-literal=password='S!B\*d$zDsb='
  
## 参照以下yaml
apiVersion: v1
kind: Secret
metadata:
  name: dev-db-secret  
data:
  password: UyFCXCpkJHpEc2I9  ## base64编码了一下
  username: ZGV2dXNlcg==


#### 2、使用文件内容
echo -n 'admin' > ./username.txt
echo -n '1f2d1e2e67df' > ./password.txt

kubectl create secret generic db-user-pass \
  --from-file=./username.txt \
  --from-file=./password.txt



# 默认密钥名称是文件名。 你可以选择使用 --from-file=[key=]source 来设置密钥名称。如下
kubectl create secret generic db-user-pass-02 \
  --from-file=un=./username.txt \
  --from-file=pd=./password.txt
## 使用yaml
dev-db-secret yaml内容如下
```

![img](https://cdn.nlark.com/yuque/0/2021/png/12365259/1639578445559-18cd91bc-3dd4-461c-92b8-9005e8e8fb0e.png)

1. 获取Secret内容

```yaml
kubectl get secret dev-db-secret -o jsonpath='{.data}'
```

#### 使用Secret

##### 环境变量引用

```shell
用这个查看进行编写
[root@k8smaster ~]# kubectl explain pod.spec.containers
# secret保存在集群的etcd里面
apiVersion: v1
kind: Pod
metadata:
  name: "pod-secret"
  namespace: default
  labels:
    app: "pod-secret"
spec:
  containers:
  - name: pod-secret
    image: "busybox"
    command: ["/bin/sh","-c","sleep 3600"]  ## echo $My_USR
    resources:
       limits:
         cpu: 10m ###  1核代表1000m
       requests: 
         cpu: 5m
    env:
    - name: MY_USR  ### 容器里的环境变量名字
      valueFrom:
        secretKeyRef: ## secret的内容
          name:  dev-db-secret #指定secret名字
          key: username  ### 自动base64解码
    - name: POD_NAME
      valueFrom:
        fieldRef: ## 属性引用
          fieldPath: metadata.name  ## 取出资源对象信息
    - name: POD_LIMIT_MEM
      valueFrom:
        resourceFieldRef:
          containerName: pod-secret  ## 取出指定容器的相关资源值
          resource: limits.cpu
```

环境变量引用的方式不会被自动更新

##### 挂卷挂载

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
```

挂载方式的secret 在secret变化的时候会自动更新**（子路径引用除外）**

如果Secret以挂载的方式给容器

- secret里面的所有key都是文件名。内容就是文件内容
- 挂载的方式会自动更新文件内容
- 反向更新？？？？

- - 挂载出来的文件是只读的
  - 容器内volumeMounts调整readOnly权限
  - 容器内的root并不是容器外的root。
  - Pod安全性。超级权限穿透。docker run --privilaged.
  - Pod里面fsGroup ,runAsUser

```yaml
# secret保存在集群的etcd里面
## kubectl create secret 
apiVersion: v1
kind: Pod
metadata:
  name: "pod-secret-volume"
  namespace: default
  labels:
    app: "pod-secret-volume"
spec:
  volumes: ## 指定挂载详情
  - name: app
    secret: 
      defaultMode: 0777 ### 修改默认的权限模式
      secretName: db-user-pass  ### secret里面所有的key全部挂出来，
      items:
      - key: password.txt  ## db-user-pass里面的password.txt这个key的内容挂出来
        path: pwd.md ### 默认secret里面数据的key是作为文件名的。 path就是相当于我们自己指定一个文件名

  containers:
  - name: pod-secret-volume
    image: "busybox"
    command: ["/bin/sh","-c","sleep 3600"]  ## echo $My_USR
    resources:
       limits:
         cpu: 10m ###  1核代表1000m
       requests: 
         cpu: 5m
    volumeMounts:
    - name: app
      mountPath: /app  ## 容器内的app文件夹是集群中 secret名叫db-user-pass的内容
      readOnly: false
kubectl get secret db-user-pass -oyaml
[root@k8smaster ~]# kubectl explain pod.spec.containers.volumeMounts
[root@k8smaster ~]# kubectl explain pod.spec.volumes
#更新
 kubectl edit secret db-user-pass
```

挂载方式的secret 在secret变化的时候会自动更新**（子路径引用除外）**

如果Secret以挂载的方式给容器

- secret里面的所有key都是文件名。内容就是文件内容
- 挂载的方式会自动更新文件内容
- 反向更新？？？？

- - 挂载出来的文件是只读的
  - 容器内volumeMounts调整readOnly权限
  - 容器内的root并不是容器外的root。
  - Pod安全性。超级权限穿透。docker run --privilaged.
  - Pod里面fsGroup ,runAsUser

## ConfigMap

对于应用的可变配置在 Kubernetes 中是通过一个 ConfigMap 资源对象来实现的，我们知道许多应用经常会有从配置文件、命令行参数或者环境变量中读取一些配置信息的需求，这些配置信息我们肯定不会直接写死到应用程序中去的，比如你一个应用连接一个 redis 服务，下一次想更换一个了的，还得重新去修改代码，重新制作一个镜像，这肯定是不可取的，而 ConfigMap 就给我们提供了向容器中注入配置信息的能力，不仅可以用来保存单个属性，还可以用来保存整个配置文件，比如我们可以用来配置一个 redis 服务的访问地址，也可以用来保存整个 redis 的配置文件。接下来我们就来了解下 ConfigMap 这种资源对象的使用方法。





### 创建

ConfigMap 资源对象使用 key-value 形式的键值对来配置数据，这些数据可以在 Pod 里面使用，如下所示的资源清单：

可以使用命令查看内容

```shell
kubectl explain cm
kind: ConfigMap
apiVersion: v1
metadata:
  name: cm-demo
  namespace: default
data:
  data.1: hello
  data.2: world
  config: |
    property.1=value-1
    property.2=value-2
    property.3=value-3
```

其中配置数据在 data 属性下面进行配置，前两个被用来保存单个属性，后面一个被用来保存一个配置文件。



我们可以看到 config 后面有一个竖线符 |，这在 yaml 中表示保留换行，每行的缩进和行尾空白都会被去掉，而额外的缩进会被保留。

```yaml
lines: |
  我是第一行
  我是第二行
    我是吴彦祖
      我是第四行
  我是第五行

# JSON
{"lines": "我是第一行\n我是第二行\n  我是吴彦祖\n     我是第四行\n我是第五行"}
```

除了竖线之外还可以使用 > 右尖括号，用来表示折叠换行，只有空白行才会被识别为换行，原来的换行符都会被转换成空格。

```yaml
lines: >
  我是第一行
  我也是第一行
  我仍是第一行
  我依旧是第一行

  我是第二行
  这么巧我也是第二行

# JSON
{"lines": "我是第一行 我也是第一行 我仍是第一行 我依旧是第一行\n我是第二行 这么巧我也是第二行"}
```

除了这两个指令之外，我们还可以使用竖线和加号或者减号进行配合使用，+ 表示保留文字块末尾的换行，- 表示删除字符串末尾的换行。

```yaml
value: |
  hello

# {"value": "hello\n"}

value: |-
  hello

# {"value": "hello"}

value: |+
  hello

# {"value": "hello\n\n"} (有多少个回车就有多少个\n)
```

当然同样的我们可以使用kubectl create -f xx.yaml来创建上面的 ConfigMap 对象，但是如果我们不知道怎么创建 ConfigMap 的话，不要忘记 kubectl 是我们最好的帮手，可以使用kubectl create configmap -h来查看关于创建 ConfigMap 的帮助信息：

```shell
Examples:
  # Create a new configmap named my-config based on folder bar
  kubectl create configmap my-config --from-file=path/to/bar

  # Create a new configmap named my-config with specified keys instead of file basenames on disk
  kubectl create configmap my-config --from-file=key1=/path/to/bar/file1.txt --from-file=key2=/path/to/bar/file2.txt

  # Create a new configmap named my-config with key1=config1 and key2=config2
  kubectl create configmap my-config --from-literal=key1=config1 --from-literal=key2=config2
```

我们可以看到可以从一个给定的目录来创建一个 ConfigMap 对象，比如我们有一个 testcm 的目录，该目录下面包含一些配置文件，redis 和 mysql 的连接信息，如下：



```shell
➜  ~ ls testcm
redis.conf
mysql.conf

➜  ~ cat testcm/redis.conf
host=127.0.0.1
port=6379

➜  ~ cat testcm/mysql.conf
host=127.0.0.1
port=3306
```

然后我们就可以使用 from-file 关键字来创建包含这个目录下面所以配置文件的 ConfigMap：



```shell
➜  ~ kubectl create configmap cm-demo1 --from-file=testcm
configmap "cm-demo1" created
```

其中 from-file 参数指定在该目录下面的所有文件都会被用在 ConfigMap 里面创建一个键值对，键的名字就是文件名，值就是文件的内容。创建完成后，同样我们可以使用如下命令来查看 ConfigMap 列表：

```shell
➜  ~ kubectl get configmap
NAME       DATA      AGE
cm-demo1   2         17s
```

可以看到已经创建了一个 cm-demo1 的 ConfigMap 对象，然后可以使用 describe 命令查看详细信息：

```yaml
➜  ~ kubectl describe configmap cm-demo1
Name:         cm-demo1
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
mysql.conf:
----
host=127.0.0.1
port=3306

redis.conf:
----
host=127.0.0.1
port=6379

Events:  <none>
```

可以看到两个 key 是 testcm 目录下面的文件名称，对应的 value 值就是文件内容，这里值得注意的是如果文件里面的配置信息很大的话，describe 的时候可能不会显示对应的值，要查看完整的键值，可以使用如下命令：

```yaml
➜  ~ kubectl get configmap cm-demo1 -o yaml
apiVersion: v1
data:
  mysql.conf: |
    host=127.0.0.1
    port=3306
  redis.conf: |
    host=127.0.0.1
    port=6379
kind: ConfigMap
metadata:
  creationTimestamp: 2018-06-14T16:24:36Z
  name: cm-demo1
  namespace: default
  resourceVersion: "3109975"
  selfLink: /api/v1/namespaces/default/configmaps/cm-demo1
  uid: 6e0f4d82-6fef-11e8-a101-525400db4df7
```

除了通过文件目录进行创建，我们也可以使用指定的文件进行创建 ConfigMap，同样的，以上面的配置文件为例，我们创建一个 redis 的配置的一个单独 ConfigMap 对象：



```yaml
➜  ~ kubectl create configmap cm-demo2 --from-file=testcm/redis.conf
configmap "cm-demo2" created
➜  ~ kubectl get configmap cm-demo2 -o yaml
apiVersion: v1
data:
  redis.conf: |
    host=127.0.0.1
    port=6379
kind: ConfigMap
metadata:
  creationTimestamp: 2018-06-14T16:34:29Z
  name: cm-demo2
  namespace: default
  resourceVersion: "3110758"
  selfLink: /api/v1/namespaces/default/configmaps/cm-demo2
  uid: cf59675d-6ff0-11e8-a101-525400db4df7
```

可以看到一个关联 redis.conf 文件配置信息的 ConfigMap 对象创建成功了，另外值得注意的是 --from-file 这个参数可以使用多次，比如我们这里使用两次分别指定 redis.conf 和 mysql.conf 文件，就和直接指定整个目录是一样的效果了。

另外，通过帮助文档我们可以看到我们还可以直接使用字符串进行创建，通过 --from-literal 参数传递配置信息，同样的，这个参数可以使用多次，格式如下：

```yaml
➜  ~ kubectl create configmap cm-demo3 --from-literal=db.host=localhost --from-literal=db.port=3306
configmap "cm-demo3" created
➜  ~ kubectl get configmap cm-demo3 -o yaml
apiVersion: v1
data:
  db.host: localhost
  db.port: "3306"
kind: ConfigMap
metadata:
  creationTimestamp: 2018-06-14T16:43:12Z
  name: cm-demo3
  namespace: default
  resourceVersion: "3111447"
  selfLink: /api/v1/namespaces/default/configmaps/cm-demo3
  uid: 06eeec7e-6ff2-11e8-a101-525400db4df7
```

### 使用

ConfigMap 创建成功了，那么我们应该怎么在 Pod 中来使用呢？我们说 ConfigMap 这些配置数据可以通过很多种方式在 Pod 里使用，主要有以下几种方式：

- 设置环境变量的值
- 在容器里设置命令行参数
- 在数据卷里面挂载配置文件

首先，我们使用 ConfigMap 来填充我们的环境变量，如下所示的 Pod 资源对象：



```yaml
apiVersion: v1
kind: Pod
metadata:
  name: testcm1-pod
spec:
  containers:
    - name: testcm1
      image: busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: cm-demo3
              key: db.host
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: cm-demo3
              key: db.port
      envFrom:
        - configMapRef:
            name: cm-demo1
```

这个 Pod 运行后会输出如下所示的信息：



```yaml
➜  ~ kubectl logs testcm1-pod
......
DB_HOST=localhost
DB_PORT=3306
mysql.conf=host=127.0.0.1
port=3306
redis.conf=host=127.0.0.1
port=6379
......
```

可以看到 DB_HOST 和 DB_PORT 都已经正常输出了，另外的环境变量是因为我们这里直接把 cm-demo1 给注入进来了，所以把他们的整个键值给输出出来了，这也是符合预期的。

另外我们也可以使用 ConfigMap来设置命令行参数，ConfigMap 也可以被用来设置容器中的命令或者参数值，如下 Pod:



```yaml
apiVersion: v1
kind: Pod
metadata:
  name: testcm2-pod
spec:
  containers:
    - name: testcm2
      image: busybox
      command: [ "/bin/sh", "-c", "echo $(DB_HOST) $(DB_PORT)" ]
      env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: cm-demo3
              key: db.host
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: cm-demo3
              key: db.port
```

运行这个 Pod 后会输出如下信息：



```shell
➜  ~ kubectl logs testcm2-pod
localhost 3306
```

另外一种是非常常见的使用 ConfigMap 的方式：通过**数据卷**使用，在数据卷里面使用 ConfigMap，就是将文件填入数据卷，在这个文件中，键就是文件名，键值就是文件内容，如下资源对象所示：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: testcm3-pod
spec:
  volumes:
    - name: config-volume
      configMap:
        name: cm-demo2
  containers:
    - name: testcm3
      image: busybox
      command: [ "/bin/sh", "-c", "cat /etc/config/redis.conf" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
```

运行这个 Pod 的，查看日志：



```shell
➜  ~ kubectl logs testcm3-pod
host=127.0.0.1
port=6379
```

当然我们也可以在 ConfigMap 值被映射的数据卷里去控制路径，如下 Pod 定义：



```yaml
apiVersion: v1
kind: Pod
metadata:
  name: testcm4-pod
spec:
  volumes:
    - name: config-volume
      configMap:
        name: cm-demo1
        items:
        - key: mysql.conf
          path: path/to/msyql.conf
  containers:
    - name: testcm4
      image: busybox
      command: [ "/bin/sh","-c","cat /etc/config/path/to/msyql.conf" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
```

运行这个Pod的，查看日志：



```shell
➜  ~ kubectl logs testcm4-pod
host=127.0.0.1
port=3306
```

另外需要注意的是，当 ConfigMap 以数据卷的形式挂载进 Pod 的时，这时更新 ConfigMap（或删掉重建ConfigMap），Pod 内挂载的配置信息会热更新。这时可以增加一些监测配置文件变更的脚本，然后重加载对应服务就可以实现应用的热更新。

使用注意

只有通过 Kubernetes API 创建的 Pod 才能使用 ConfigMap，其他方式创建的（比如静态 Pod）不能使用；ConfigMap 文件大小限制为 1MB（ETCD 的要求）。



### ConfigMap实战

- ConfigMap保存的内容就是普通的明文文本，不会编码。
- ConfigMap 来将你的配置数据和应用程序代码分开。
- ConfigMap 是一种 API 对象，用来将非机密性的数据保存到键值对中。使用时， [Pods](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 可以将其用作环境变量、命令行参数或者存储卷中的配置文件。

```shell
kubectl create configMap --help

  kubectl create configmap my-config --from-literal=key1=config1 --from-literal=key2=config2

[root@k8smaster ~]# kubectl get cm

[root@k8smaster ~]# kubectl get cm -oyaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # 类属性键；每一个键都映射到一个简单的值
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # 类文件键
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
```

你可以使用四种方式来使用 ConfigMap 配置 Pod 中的容器：

1. 在容器命令和参数内
2. 容器的环境变量
3. 在只读卷里面添加一个文件，让应用来读取
4. 编写代码在 Pod 中运行，使用 Kubernetes API 来读取 ConfigMap

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      env:
        # 定义环境变量
        - name: PLAYER_INITIAL_LIVES # 请注意这里和 ConfigMap 中的键名是不一样的
          valueFrom:
            configMapKeyRef:
              name: game-demo           # 这个值来自 ConfigMap
              key: player_initial_lives # 需要取值的键
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
      volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:
    # 你可以在 Pod 级别设置卷，然后将其挂载到 Pod 内的容器中
    - name: config
      configMap:
        # 提供你想要挂载的 ConfigMap 的名字
        name: game-demo
        # 来自 ConfigMap 的一组键，将被创建为文件
        items:
        - key: "game.properties"  ### 只把cm的部分key挂载出来成文件
          path: "game.properties" ### 指定在容器中保存的文件名
        - key: "user-interface.properties"
          path: "user-interface.properties"
```

**ConfigMap的修改，可以触发挂载文件的自动更新**

注意：：：：：：：：

CM和Secret使用子路径挂载的话是无法热更新。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # 类属性键；每一个键都映射到一个简单的值
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"
  # 类文件键    aa \ 
  game.properties: |  ## 多行内容写法
    enemy.types=aliens,monsters
    player.maximum-lives=5   
 
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
---  现部署上面再部署下面
apiVersion: v1
kind: Pod
metadata:
  name: "cm-pod"
  namespace: default
  labels:
    app: "cm-pod"
spec:
  volumes:
  - name: my-cm
    configMap: 
      name: game-demo ## 指定cm的名字
      # items: 
      #  - 
  containers:
  - name: cm-pod
    image: "busybox"
    env:
    - name: PLAY_LIVES
      valueFrom:
        configMapKeyRef:  ##来自于cm
          name: game-demo
          key: player_initial_lives
    volumeMounts:
    - name: my-cm
      mountPath: /app  ## 容器内的app文件夹是集群中 secret名叫db-user-pass的内容
```

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669303464365-17fbf666-1bfd-46d1-9eb1-eb3966b831e6.png)



## ServiceAccount

ServiceAccount 主要是用于解决 Pod 在集群中的身份认证问题的。认证使用的授权信息其实就是利用前面我们讲到的一个类型为 kubernetes.io/service-account-token 进行管理的。

### 介绍

ServiceAccount 是命名空间级别的，每一个命名空间创建的时候就会自动创建一个名为 default 的 ServiceAccount 对象:

```plain
$ kubectl create ns kube-test
namespace/kube-test created
$ kubectl get sa -n kube-test
NAME      SECRETS   AGE
default   1         9s
$ kubectl get secret -n kube-test
NAME                  TYPE                                  DATA   AGE
default-token-vn4tr   kubernetes.io/service-account-token   3      2m27s
```

这个 ServiceAccount 会自动关联到一个 Secret 对象上：

```plain
$ kubectl get sa default -n kube-test  -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2019-11-23T04:19:47Z"
  name: default
  namespace: kube-test
  resourceVersion: "4297522"
  selfLink: /api/v1/namespaces/kube-test/serviceaccounts/default
  uid: 75b3314b-e949-4f7b-9450-9bcd89c8c972
secrets:
- name: default-token-vn4tr
```

这个 Secret 对象是 ServiceAccount 控制器自动创建的，我们可以查看这个关联的 Secret 对象信息：

```shell
$ kubectl get secret default-token-vn4tr -n kube-test -o yaml
apiVersion: v1
data:
  ca.crt: LS0tLS...
  namespace: a3ViZS10ZXN0
  token: ZXlKaG...
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: default
    kubernetes.io/service-account.uid: 75b3314b-e949-4f7b-9450-9bcd89c8c972
  creationTimestamp: "2019-11-23T04:19:47Z"
  name: default-token-vn4tr
  namespace: kube-test
  resourceVersion: "4297521"
  selfLink: /api/v1/namespaces/kube-test/secrets/default-token-vn4tr
  uid: e3e60f95-f255-471b-a6c0-600a3c0ee53a
type: kubernetes.io/service-account-token
```

在 data 区域我们可以看到有3个信息：

- ca.crt：用于校验服务端的证书信息
- namespace：表示当前管理的命名空间
- token：用于 Pod 身份认证的 Token

前面我们也提到了默认情况下当前 namespace 下面的 Pod 会默认使用 default 这个 ServiceAccount，对应的 Secret 会自动挂载到 Pod 的 /var/run/secrets/kubernetes.io/serviceaccount/ 目录中，这样我们就可以在 Pod 里面获取到用于身份认证的信息了。

### 实现原理

实际上这个自动挂载过程是在 Pod 创建的时候通过 Admisson Controller（准入控制器） 来实现的，关于准入控制器的详细信息我们会在后面的安全章节中和大家继续学习。

**Admisson Controller**

Admission Controller（准入控制）是 Kubernetes API Server 用于拦截请求的一种手段。Admission 可以做到对请求的资源对象进行校验，修改，Pod 创建时 Admission Controller 会根据指定的的 ServiceAccount（默认的 default）把对应的 Secret 挂载到容器中的固定目录下 /var/run/secrets/kubernetes.io/serviceaccount/。

然后当我们在 Pod 里面访问集群的时候，就可以默认利用挂载到 Pod 内部的 token 文件来认证 Pod 的身份，ca.crt 则用来校验服务端。在 Pod 中访问 Kubernetes 集群的一个典型的方式如下图所示：

![img](https://cdn.nlark.com/yuque/0/2023/jpeg/12365259/1704035557094-0e79e835-f278-4bd1-8deb-cd71077eb3b6.jpeg)

代码中我们指定了 ServiceAccount 背后的 Secret 挂载到 Pod 里面的两个文件：token 和 ca.crt，然后通过环境变量获取到 APIServer 的访问地址（前面我们提到过会把 Service 信息通过环境变量的方式注入到 Pod 中），然后通过 ca.cart 校验服务端是否可信，最后服务端会根据我们提供的 token 文件对 Pod 进行认证。

Pod 身份被认证合法过后，具体具有哪些资源的访问权限，就需要通过后面的 RBAC 来进行声明了。所以我们在学习 Kubernetes 的权限认证的时候需要把整个流程弄清楚，ServiceAccount 是干嘛的？为什么还需要 RBAC？这些都是上下文关联的。

# 临时存储

## 几种临时存储

Kubernetes 为了不同的目的，支持几种不同类型的临时卷：

- [emptyDir](https://kubernetes.io/zh/docs/concepts/storage/volumes/#emptydir)： Pod 启动时为空，存储空间来自本地的 kubelet 根目录（通常是根磁盘）或内存
- [configMap](https://kubernetes.io/zh/docs/concepts/storage/volumes/#configmap)、 [downwardAPI](https://kubernetes.io/zh/docs/concepts/storage/volumes/#downwardapi)、 [secret](https://kubernetes.io/zh/docs/concepts/storage/volumes/#secret)： 将不同类型的 Kubernetes 数据注入到 Pod 中
- [CSI 临时卷](https://kubernetes.io/zh/docs/concepts/storage/volumes/#csi-ephemeral-volumes)： 类似于前面的卷类型，但由专门[支持此特性](https://kubernetes-csi.github.io/docs/drivers.html) 的指定 [CSI 驱动程序](https://github.com/container-storage-interface/spec/blob/master/spec.md)提供
- [通用临时卷](https://kubernetes.io/zh/docs/concepts/storage/ephemeral-volumes/#generic-ephemeral-volumes)： 它可以由所有支持持久卷的存储驱动程序提供

## emptyDir

- 当 Pod 分派到某个 Node 上时，emptyDir 卷会被创建
- 在 Pod 在该节点上运行期间，卷一直存在。
- 卷最初是空的。 
- 尽管 Pod 中的容器挂载 emptyDir 卷的路径可能相同也可能不同，这些容器都可以读写 emptyDir 卷中相同的文件。 
- 当 Pod 因为某些原因被从节点上删除时，emptyDir 卷中的数据也会被永久删除。
- 存储空间来自本地的 kubelet 根目录（通常是根磁盘）或内存

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "multi-container-pod"
  namespace: default
  labels:
    app: "multi-container-pod"
spec:
  volumes:    ### 以后见到的所有名字 都应该是一个合法的域名方式
  - name: nginx-vol
    emptyDir: {}  ### docker匿名挂载，外部创建一个位置  /abc
  containers:  ## kubectl exec -it podName  -c nginx-container（容器名）-- /bin/sh
  - name: nginx-container
    image: "nginx"
    volumeMounts:  #声明卷挂载  -v
      - name: nginx-vol
        mountPath: /usr/share/nginx/html
  - name: content-container
    image: "alpine"
    command: ["/bin/sh","-c","while true;do sleep 1; date > /app/index.html;done;"]
    volumeMounts: 
      - name: nginx-vol
        mountPath: /app
```

Pod里面是容器，容器无论怎样重启不会导致卷丢失。

Pod真正被删除、重新拉起（比如Pod超出资源限制，kubelet就会杀死Pod，如果要保证副本就重新拉起，此时临时卷就没有）。

## 扩展-hostPath

https://kubernetes.io/zh/docs/concepts/storage/volumes/#hostpath

![img](https://cdn.nlark.com/yuque/0/2021/png/12365259/1639580043587-2239117e-06db-4357-a837-684e2a6968a4.png)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      # 宿主上目录位置
      path: /data
      # 此字段为可选
      type: Directory
apiVersion: v1
kind: Pod
metadata:
  name: test-webserver
spec:
  containers:
  - name: test-webserver
    image: k8s.gcr.io/test-webserver:latest
    volumeMounts:
    - mountPath: /var/local/aaa
      name: mydir
    - mountPath: /var/local/aaa/1.txt
      name: myfile
  volumes:
  - name: mydir
    hostPath:
      # 确保文件所在目录成功创建。
      path: /var/local/aaa
      type: DirectoryOrCreate
  - name: myfile
    hostPath:
      path: /var/local/aaa/1.txt
      type: FileOrCreate
```

典型应用

解决容器时间问题

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busy-box-test
  namespace: default
spec:
  containers:
  - name: busy-box-test
    image: busybox
    imagePullPolicy: IfNotPresent
    volumeMounts: ## 描述容器想把自己的哪个路径进行挂载
    - name: localtime
      mountPath: /etc/localtime  ## 挂到容器的这个位置
  volumes:  ## 描述每个volumeMounts到底该怎么挂载
    - name: localtime
      hostPath:  ## 主机的这个文件
        path: /usr/share/zoneinfo/Asia/Shanghai  ## 上海时区
apiVersion: v1
kind: Pod
metadata:
  name: "pod-time"
  namespace: default
  labels:
    app: "pod-time"
spec:
  containers:
  - name: pod-time
    image: "busybox"
    command: ["sleep","60000"]
    volumeMounts: ## 描述容器想把自己的哪个路径进行挂载
    - name: localtime
      mountPath: /etc/localtime  ## 挂到容器的这个位置
      # subPath: localtime  ## /etc/localtime
  volumes:  ## 描述每个volumeMounts到底该怎么挂载
    - name: localtime
      hostPath:  ## 主机的这个文件  
        path: /usr/share/zoneinfo/Asia/Shanghai
        type: Directory  ### 到底是什么。文件/文件夹 .....
```

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669305204352-84b3942b-8009-4c17-9294-578adb3578f7.png)

```shell
apiVersion: v1
kind: Pod
metadata:
  name: "pod-time"
  namespace: default
  labels:
    app: "pod-time"
spec:
  containers:
  - name: pod-time
    image: "busybox"
    command: ["sleep","60000"]
    volumeMounts: ## 描述容器想把自己的哪个路径进行挂载
    - name: localtime
      mountPath: /etc/localtime  ## 挂到容器的这个位置
      # subPath: localtime  ## /etc/localtime
  volumes:  ## 描述每个volumeMounts到底该怎么挂载
    - name: localtime
      hostPath:  ## 主机的这个文件  
        path: /usr/share/zoneinfo/Asia/Shanghai
        type: Directory  ### 到底是什么。文件/文件夹 .....
```

# 持久化

## VOLUME

### 基础

![img](https://cdn.nlark.com/yuque/0/2021/png/12365259/1639580160251-595f9cad-fbbb-4687-a7a6-f8d8236da6b3.png)

- Kubernetes 支持很多类型的卷。 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 可以同时使用任意数目的卷类型
- 临时卷类型的生命周期与 Pod 相同，但持久卷可以比 Pod 的存活期长
- 当 Pod 不再存在时，Kubernetes 也会销毁临时卷；
- Kubernetes 不会销毁 持久卷。
- 对于给定 Pod 中**任何类型的卷**，在容器重启期间数据都不会丢失。
- 使用卷时, 在 .spec.volumes 字段中设置为 Pod 提供的卷，并在 .spec.containers[*].volumeMounts 字段中声明卷在容器中的挂载位置。

[支持的卷类型](https://kubernetes.io/zh/docs/concepts/storage/volumes/#volume-types)

### 使用NFS

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669330771072-0be515aa-3e67-45ec-a5bf-66b2f876755c.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669330811970-6ce3cf74-d7e5-4f69-ab72-8c64d7166533.png)

#### 安装NFS

```yaml
# 在任意机器。
yum install -y nfs-utils
#执行命令 vi /etc/exports，创建 exports 文件，文件内容如下：
#所有的都支持
echo "/nfs/data/ *(insecure,rw,sync,no_root_squash)" > /etc/exports
这个ip范围内
echo "/nfs/data/ 172.18.0.0/20(insecure,rw,sync,no_root_squash)" > /etc/exports

#/nfs/data  172.26.248.0/20(rw,no_root_squash)

# 执行以下命令，启动 nfs 服务;创建共享目录
mkdir -p /nfs/data
systemctl enable rpcbind
systemctl enable nfs-server
systemctl start rpcbind
systemctl start nfs-server
exportfs -r
#检查配置是否生效
exportfs
# 输出结果如下所示
/nfs/data /nfs/data
```

客户端创建

#### 扩展-NFS文件同步

```shell
#服务器端防火墙开放111、662、875、892、2049的 tcp / udp 允许，否则远端客户无法连接。
#安装客户端工具
yum install -y nfs-utils


#执行以下命令检查 nfs 服务器端是否有设置共享目录
# showmount -e $(nfs服务器的IP)
showmount -e 172.18.15.116
# 输出结果如下所示
Export list for 172.18.15.116
/nfs/data *

#执行以下命令挂载 nfs 服务器上的共享目录到本机路径 /root/nfsmount
mkdir /root/nfsmount
# mount -t nfs $(nfs服务器的IP):/root/nfs_root /root/nfsmount
#高可用备份的方式
mount -t nfs 172.18.15.116:/nfs/data /root/nfsmount
# 写入一个测试文件
echo "hello nfs server" > /root/nfsmount/test.txt

#在 nfs 服务器上执行以下命令，验证文件写入成功
cat /root/nfsmount/test.txt
```



#### VOLUME进行挂载测试

```yaml
#测试Pod直接挂载NFS了
apiVersion: v1
kind: Pod
metadata:
  name: vol-nfs
  namespace: default
spec:
  containers:
  - name: myapp
    image: nginx
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html/
  volumes:
  - name: html
    nfs:
      path: /nfs/data   #1000G
      server: 自己的nfs服务器地址
apiVersion: v1
kind: Pod
metadata:
  name: pod-nfc
  labels:
    app: pod-nfc
spec:
  volumes:
    - name: html
      nfs:
        path: /nfs/data/nginx/
        server: 172.18.15.116
  containers:
    - name: pod-nfc
      image: nginx
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - mountPath: /usr/share/nginx/html/
          name: html
  restartPolicy: Always
```

第一个demo

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "pod-nfs-02"
  namespace: default
  labels:
    app: "pod-nfs-02"
spec:
  containers:
  - name: pod-nfs-02
    image: "nginx"
    ports:
    - containerPort:  80
      name:  http
    volumeMounts:
    - name: localtime
      mountPath: /etc/localtime
    - name: html
      mountPath: /usr/share/nginx/html  ## /一定会是文件夹。不带/请指定好类型
      # type: DirectoryOrCreate
  volumes: 
    - name: localtime
      hostPath:  
        path: /usr/share/zoneinfo/Asia/Shanghai
    - name: html
      nfs:  ## 使用nfs存储系统
        server: 10.170.11.8  ## 没type
        path: /nfs/data/abc  ### abc文件夹提前创建
```



## PV&PVC&StorageClass

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669331353692-e2f0df75-1257-49dd-b8e6-06c6e56b71a7.png)

```yaml
persistentVolumeClaim:  ## 持久卷申请 关联一个 PV（PersistentVolume）持久卷
        # xxx，  8G，ssd。  k8s给你指定的持久化空间  PVC --- PV
[root@k8smaster ~]# kubectl get pv
kubectl get pv pv-volume-1g -oyaml
kubectl get pvc
```

案例:

创建一个存储类

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-nfs-storage
  labels:
    type: local
spec:
  storageClassName: my-nfs-storage
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  nfs:  ## 使用nfs存储系统
    server: 172.18.15.116 ## 没type
    path: /nfs/data/  ### abc文件夹提前创建
```

创建一个pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "nginx-pvc"
  namespace: default
  labels:
    app: "nginx-pvc"
spec:
  containers:
    - name: nginx-pvc
      image: "nginx"
      ports:
        - containerPort:  80
          name:  http
      volumeMounts:
        - name: localtime
          mountPath: /etc/localtime
        - name: html
          mountPath: /usr/share/nginx/html
  volumes:
    - name: localtime
      hostPath:
        path: /usr/share/zoneinfo/Asia/Shanghai
    - name: html
      persistentVolumeClaim:
        claimName:  nginx-pvc
  restartPolicy: Always
--- 申请券
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
  namespace: default
  labels:
    app: nginx-pvc
spec:
  storageClassName: "my-nfs-storage"  ## 存储类的名字
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Mi
```

### 基础概念

- **存储的管理**是一个与**计算实例的管理**完全不同的问题。
- PersistentVolume 子系统为用户 和管理员提供了一组 API，将存储如何供应的细节从其如何被使用中抽象出来。 
- 为了实现这点，我们引入了两个新的 API 资源：PersistentVolume 和 PersistentVolumeClaim。

**持久卷（PersistentVolume ）：**

- 持久卷（PersistentVolume，PV）是集群中的一块存储，可以由管理员事先供应，或者 使用[存储类（Storage Class）](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/)来动态供应。
- 持久卷是集群资源，就像节点也是集群资源一样。PV 持久卷和普通的 Volume 一样，也是使用 卷插件来实现的，只是它们拥有独立于使用他们的Pod的生命周期。
- 此 API 对象中记述了存储的实现细节，无论其背后是 NFS、iSCSI 还是特定于云平台的存储系统。

**持久卷申请（PersistentVolumeClaim，PVC）：**

- 表达的是用户对存储的请求
- 概念上与 Pod 类似。 Pod 会耗用节点资源，而 PVC 申领会耗用 PV 资源。
- Pod 可以请求特定数量的资源（CPU 和内存）；同样 PVC 申领也可以请求特定的大小和访问模式 （例如，可以要求 PV 卷能够以 ReadWriteOnce、ReadOnlyMany 或 ReadWriteMany 模式之一来挂载，参见[访问模式](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#access-modes)）。

**存储类（Storage Class）**:

- 尽管 PersistentVolumeClaim 允许用户消耗抽象的存储资源，常见的情况是针对不同的 问题用户需要的是具有不同属性（如，性能）的 PersistentVolume 卷。
- 集群管理员需要能够提供不同性质的 PersistentVolume，并且这些 PV 卷之间的差别不 仅限于卷大小和访问模式，同时又不能将卷是如何实现的这些细节暴露给用户。
- 为了满足这类需求，就有了 *存储类（StorageClass）* 资源。

![img](https://cdn.nlark.com/yuque/0/2021/png/12365259/1639662909871-83a115ee-c4b1-4375-b195-bd76c4351a3d.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/12365259/1639663049867-3fc47455-d54b-46a0-8fba-a64c4e0be282.png)



![img](https://cdn.nlark.com/yuque/0/2021/png/12365259/1639663082297-c3d92f04-7df2-4b6f-a9be-59c96fd83b58.png)

### 实战

https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-persistent-volume-storage/

只要PVC有：

- PVC先是Pending
- PV被创建以后（正好是PVC可用的），PVC自动绑定到这个PV卷
- PVC可以被提前创建并和PV绑定。以后Pod只需要关联PVC即可
- Pod删除，PVC还在吗？

- - pvc并不会影响。**手动删除pvc。**

- - - 按照以前原则。同一个资源的所有关联东西都应该写在一个文件中，方便管理

- - PVC删除会不会影响PV？

- - - 要看pv的回收策略。比如nfs。回收策略是Recycle。pv不会删除，内容会被清空，pva重新变为Available

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "nginx-pvc"
  namespace: default
  labels:
    app: "nginx-pvc"
spec:
  containers:
    - name: nginx-pvc
      image: "nginx"
      ports:
        - containerPort:  80
          name:  http
      volumeMounts:
        - name: localtime
          mountPath: /etc/localtime
        - name: html
          mountPath: /usr/share/nginx/html
  volumes:
    - name: localtime
      hostPath:
        path: /usr/share/zoneinfo/Asia/Shanghai
    - name: html
      persistentVolumeClaim:
        claimName:  nginx-pvc
  restartPolicy: Always
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
  namespace: default
  labels:
    app: nginx-pvc
spec:
  storageClassName: my-nfs-storage  ## 存储类的名字
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50m
```

### 细节

#### 访问模式

https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#access-modes



#### 回收策略

https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#reclaim-policy

目前的回收策略有：

- Retain -- 手动回收： pv相关的内容，自己删除。（pvc删除，pv分文不动，自己手动控制）

- - pv： pvc删除了。pod.yaml ---> pod的部署，pvc ---> 部署数据库
  - 开发删除了Pod，希望k8s重新拉起（一旦是Recycle 、Delete ）
  - Retain ：默认规则。Pod重新拉起

- - - 即使Pod以及PVC删除。但是pv会记住上次是和哪个pvc建立的绑定关系
    - 只要下次Pod继续使用这个PVC，依然能重新建立连接

- Recycle -- 基本擦除 (rm -rf /thevolume/*)：清除卷里面的内容
- Delete -- 诸如 AWS EBS、GCE PD、Azure Disk 或 OpenStack Cinder 卷这类关联存储资产也被删



目前，仅 NFS 和 HostPath 支持回收（Recycle）。 AWS EBS、GCE PD、Azure Disk 和 Cinder 卷都支持删除（Delete）。： pv自己也跟着删除



删除



```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-volume-12m-recycle
  labels:
    type: local
spec:
  persistentVolumeReclaimPolicy: Recycle  ## 回收
  storageClassName: my-nfs-storage
  capacity:
    storage: 12m
  accessModes:
    - ReadWriteOnce 
  nfs:  ## 使用nfs存储系统
    server: 10.170.11.8  ## 没type
    path: /nfs/data/recycle  ### abc文件夹提前创建
---  单独执行，先执行这个，在执行上面一截
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pv-12-recycle
  namespace: default
  labels:
    app: pv-12-recycle
spec:
  storageClassName: my-nfs-storage  ### pv分组
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 12m
```

#### 阶段

https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#phase

- Released

- - pv释放。释放了和pvc的关联关系，绑定不存在。新的不同pvc不能重新绑定上来

- Available

- - pv可用。可以和任意pvc进行绑定

每个卷会处于以下阶段（Phase）之一：

- Available（可用）-- 卷是一个空闲资源，尚未绑定到任何申领；任意人都能继续使用
- Bound（已绑定）-- 该卷已经绑定到某申领；别人不能再用
- Released（已释放）-- 所绑定的申领已被删除，但是资源尚未被集群回收；pvc被删除了，pv没有被回收

- - 自己确认这个pv没用了，自己删除pv；重新创建出这个pv
  - pv还有用，可以重新绑定

- Failed（失败）-- 卷的自动回收操作失败。

绑定了Pod的PVC，而且Pod正在运行，PVC是不能被删除的，pvc会一直等到pod删除了后再去删除

### 动态供应

![img](https://cdn.nlark.com/yuque/0/2021/png/12365259/1639663162158-c2d06aa0-1e7b-4589-872d-b9bd7b81a6cb.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669415684847-a918bf50-a4fb-436b-b32e-35e37853762b.png)

静态供应：

- 集群管理员创建若干 PV 卷。这些卷对象带有真实存储的细节信息，并且对集群 用户可用（可见）。PV 卷对象存在于 Kubernetes API 中，可供用户消费（使用）

动态供应：

- 集群自动根据PVC创建出对应PV进行使用

#### 设置nfs动态供应

https://github.com/kubernetes-retired/external-storage/tree/master/nfs-client

按照文档部署，并换成 registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/nfs-subdir-external-provisioner:v4.0.2 镜像即可



```yaml
## 创建了一个存储类
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner 
#provisioner指定一个供应商的名字。  
#必须匹配 k8s-deployment 的 env PROVISIONER_NAME的值
parameters:
  archiveOnDelete: "false"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 10.170.11.8 ## 指定自己nfs服务器地址
            - name: NFS_PATH  
              value: /nfs/data  ## nfs服务器共享的目录
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.170.11.8
            path: /nfs/data
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

申请

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "nginx-666-pvc-000"
  namespace: default
  labels:
    app: "nginx-666-pvc-000"
spec:
  containers:
  - name: nginx-666-pvc-000
    image: "nginx"
    ports:
    - containerPort:  80
      name:  http
    volumeMounts:
    - name: localtime
      mountPath: /etc/localtime
    - name: html
      mountPath: /usr/share/nginx/html
  volumes:
    - name: localtime
      hostPath:
        path: /usr/share/zoneinfo/Asia/Shanghai
    - name: html
      persistentVolumeClaim:
         claimName:  nginx-666-pvc  ### 你的申请书的名字
  restartPolicy: Always
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-666-pvc
  namespace: default
  labels:
    app: nginx-666-pvc
spec:
  storageClassName: managed-nfs-storage  ## 存储类的名字
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 70m
# ---
# apiVersion: v1
# kind: Service
# metadata:
#   name: MYAPP
#   namespace: default
# spec:
#   selector:
#     app: MYAPP
#   type: ClusterIP
#   ports:
#   - name: MYAPP
#     port: 
#     targetPort: 
#     protocol: TCP
#     nodePort: 
```

## 指定一个SC为默认驱动

```shell
kubectl patch storageclass managed-nfs-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

加上这个指定默认annotations:

​    storageclass.kubernetes.io/is-default-class: "true" #指定默认

```yaml
## 创建了一个存储类
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true" #指定默认
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner 
#provisioner指定一个供应商的名字。  
#必须匹配 k8s-deployment 的 env PROVISIONER_NAME的值
parameters:
  archiveOnDelete: "true"  ## 删除pv的时候，pv的内容是否要备份
  #### 这里可以调整供应商能力。

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/nfs-subdir-external-provisioner:v4.0.2
          # resources:
          #    limits:
          #      cpu: 10m
          #    requests:
          #      cpu: 10m
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 10.170.11.8 ## 指定自己nfs服务器地址
            - name: NFS_PATH  
              value: /nfs/data  ## nfs服务器共享的目录
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.170.11.8
            path: /nfs/data
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

申请书

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc-999
  namespace: default
  labels:
    app: nginx-pvc-999
spec:
  # storageClassName: my-nfs-storage  ## 存储类的名字
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10m
      
  不指定存储驱动，就使用默认的
```

使用动态供应做的nfs挂载的pv

支持delete回收策略了。