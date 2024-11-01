# 简介

自己写yaml 

一个应用：（博客程序，wordpress+mysql） 

- Deployment.yaml 
- Service.yaml 
- PVC.yaml 
- Ingress.yaml 
- xxxx 

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669512852304-8e43a27c-c958-4568-833e-ba9f87cffcea.png)

charts：图表 发布charts；docker发布镜像

 官网:https://helm.sh/zh/docs/intro/using_helm/

# 安装

## 用二进制版本安装

每个Helm **版本**都提供了各种操作系统的二进制版本，这些版本可以手动下载和安装。 

1. 下载 **需要的版本** https://github.com/helm/helm/releases

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669513315005-273ab5c4-3139-4ed2-8a80-e8483f46ee91.png)

1. 解压( tar -zxvf helm-v3.0.0-linux-amd64.tar.gz ) 
2. 在解压目中找到 helm 程序，移动到需要的目录中( mv linux-amd64/helm 

/usr/local/bin/helm )

# 入门概念

## 三大概念

- *Chart* 代表着 Helm 包。它包含在 Kubernetes 集群内部运行应用程序，工具或服务所需的所有资源 定义。你可以把它看作是 Homebrew formula，Apt dpkg，或 Yum RPM 在Kubernetes 中的等价物。 
- *Repository*（仓库） 是用来存放和共享 charts 的地方。它就像 Perl 的 **CPAN 档案库网络** 或是 Fedora 的 **软件包仓库**，只不过它是供 Kubernetes 包所使用的。 
- *Release* 是运行在 Kubernetes 集群中的 chart 的实例。一个 chart 通常可以在同一个集群中安装多 次。每一次安装都会创建一个新的 *release* 。以 MySQL chart为例，如果你想在你的集群中运行两个数据库，你可以安装该chart两次。每一个数据库都会拥有它自己的 *release* 和 *release name* 。

Helm 安装 *charts* 到 Kubernetes 集群中，每次安装都会创建一个新的 *release* 。你可以在 Helm 

的 chart *repositories* 中寻找新的 chart。 

```shell
helm pull bitnami/mysql
helm install -f values.yaml mysqlhaha ./
```

## Charts结构

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669513485537-b2ffd294-0f61-438a-9434-f9b64c762a99.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669513492961-5d356635-58ae-4149-b793-df0fba4ada31.png)

## 应用安装

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669513505011-4a3496a1-a8c1-44a3-8eb9-53747e79aaa5.png)

## 自定义变量值

![img](https://cdn.nlark.com/yuque/0/2022/png/12365259/1669513517146-6be83253-27c1-4a5d-aa55-5c1abbfb2c6f.png)

## 命令

```shell
helm install xx 

helm list 

helm status xx 

helm rollback xxx
```

# Mysql的小实验

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-cluster
  namespace: default
spec:
  serviceName: mysql  # 一定指定StatefulSet的serviceName
  selector:
    matchLabels:
      app: mysql 
  replicas: 2 # by default is 1 。默认也是负载均衡
  ### 自己连进这两个mysql。才能主从同步
  template:
    metadata:
      labels:
        app: mysql # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: mysql
        image: mysql:8.0.25
        securityContext:  ## 指定安全上下文
          runAsUser: 1000
          runAsGroup: 1000
        ports:
        - containerPort: 3306
          name: mysql-port
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "123456"
        - name: MYSQL_DATABASE
          value: "db_test"
        volumeMounts:
        - name: mysql-cnf
          mountPath: /etc/mysql/conf.d
        - name: mysql-data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-cnf
    spec:
      storageClassName: "managed-nfs-storage"
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
  - metadata:
      name: mysql-data
    spec:
      storageClassName: "managed-nfs-storage"
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: default
spec:
  selector:
    app: mysql
  type: NodePort 
  # type: ClusterIP
  # clusterIP: None  ## 没有集群ip，只能通过内部的dns访问 mysql-cluster-0.mysql.default
  ports:
  - name: mysql-port
    port: 3306 ## service端口
    targetPort:  3306 ## pod端口
    protocol: TCP


# ---
# apiVersion: v1
# kind: PersistentVolumeClaim
# metadata:
#   name: hahah
#   namespace: default
#   labels:
#     app: hahah
# spec:
#   storageClassName: "managed-nfs-storage"
#   accessModes:
#   - ReadWriteOnce
#   resources:
#     requests:
#       storage: 1Gi
```