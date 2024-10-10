Kubernetes 网络和负载均衡
一、Kubernetes网络
Kubernetes 网络解决四方面的问题：
一个 Pod 中的容器之间通过本地回路（loopback）通信。
集群网络在不同 pod 之间提供通信。Pod和Pod之间互通
Service 资源允许你对外暴露 Pods 中运行的应用程序，以支持来自于集群外部的访问。Service和
Pod要通
可以使用 Services 来发布仅供集群内部使用的服务。
1、k8s网络架构图
1、架构图



2、访问流程


2、网络连通原理

1、Container To Container



1.  ip netns add ns1  #添加网络名称空间
2. ls /var/run/netns #查看所有网络名词空间
3. ip netns  #查看所有网络名词空间
# Linux 将所有的进程都分配到 root network namespace，以使得进程可以访问外部网络
# Kubernetes 为每一个 Pod 都创建了一个 network namespace
2、Pod To Pod
1、同节点


2、跨节点


3、Pod-To-Service
1、Pod To Service




2、Service-To-Pod





4、Internet-To-Service

1、Pod-To-Internet






2、Internet-To-Pod（LoadBalancer -- Layer4）


3、Internet-To-Pod（Ingress-- Layer7）


二、Service
负载均衡服务。让一组Pod可以被别人进行服务发现。
Service --- >> 选择一组Pod
别人只需要访问这个Service。Service还会基于Pod的探针机制（ReadinessProbe：就绪探针）完成Pod的自动剔除和上线工作。
● Service即使无头服务。别人（Pod）不能用ip访问，但是可以用service名当成域名访问。
● Service的名字还能当成域名被Pod解析

1、基础概念

：通过集群的内部 IP 暴露服务，选择该值时服务只能够在集群内部访问。 这也是默认的 ServiceType 。
将运行在一组 Pods 上的应用程序公开为网络服务的抽象方法。
云原生服务发现
service中的type可选值如下，代表四种不同的服务发现类型 ExternalName
ClusterIP: 为当前Service分配或者不分配集群IP。负载均衡一组Pod NodePort： 外界也可以使用机器ip+暴露的NodePort端口 访问。
nodePort端口由kube-proxy开在机器上
机器ip+暴露的NodePort 流量先来到 kube-proxy LoadBalancer.

：通过每个节点上的 IP 和静态端口（ NodePort ）暴露服务。	服务会路
NodePort
由到自动创建的  ClusterIP  服务。 通过请求  <节点  IP>:<节点端口> ，你可以从集群的外部访问一个 NodePort 服务。
：使用云提供商的负载均衡器向外部暴露服务。 外部负载均衡器可以将流量路由到自动创建的 NodePort 服务和 ClusterIP 服务上。
：通过返回 CNAME 和对应值，可以将服务映射到 externalName 字段的内容
（例如，  foo.bar.example.com  ）。 无需创建任何类型代理。
1、创建简单Service


1
2
3
4
5
6
7
8
9
10
11
12
apiVersion: v1 kind: Service metadata:
name: my-service spec:
selector:
app:  MyApp
## 使用选择器选择所有Pod
# type: ClusterIP ##type很重要，不写默认是ClusterIP
ports:
- protocol: TCP port: 80
  targetPort:  9376
  Service 创建完成后，会对应一组EndPoint。可以kubectl get ep 进行查看
  type有四种，每种对应不同服务发现机制
  Servvice可以利用Pod的就绪探针机制，只负载就绪了的Pod。自动剔除没有就绪的Pod


2、创建无Selector的Service
我们可以创建Service不指定Selector
然后手动创建EndPoint，指定一组Pod地址。  此场景用于我们负载均衡其他中间件场景。

1. # 无selector的svc
2. apiVersion: v1
3. kind: Service
4. metadata:
5. name: my-service-no-selector
6. spec:
7. ports:
8. - protocol: TCP
9. name: http  ###一定注意，name可以不写，
10. ###但是这里如果写了name，那么endpoint里面的ports必须有同名name才能绑定
11. port: 80  # service 80
12. targetPort: 80  #目标80
13. ---
14. apiVersion: v1
15. kind: Endpoints
16. metadata:
17. 	name: my-service-no-selector ### ep和svc的绑定规则是：和svc同名同名称空间，port同名或同端口
18. namespace: default
19. subsets:
20. - addresses:
      21	- ip: 220.181.38.148
      22	- ip: 39.156.69.79
      23	- ip: 192.168.169.165
1. ports:
2. - port: 80
3. name: http  ## svc有name这里一定要有
4. protocol: TCP

原理：kube-proxy 在负责这个事情
https://kubernetes.io/zh/docs/concepts/services-networking/service/#virtual-ips-and-service-proxies

1. ## 实验
2. apiVersion: v1
3. kind: Service
4. metadata:
5. name: cluster-service-no-selector
6. namespace: default
7. spec:
8. ##  不选中Pod而在下面手动定义可以访问的EndPoint
9. type: ClusterIP
10. ports:
11. - name: abc
12. port: 80  ## 访问当前service 的 80
13. targetPort: 80  ## 派发到Pod的 80
14. ---
15. apiVersion: v1
16. kind: Endpoints
17. metadata:
18. name: cluster-service-no-selector  ## 和service同名
19. namespace: default
20. subsets:
21. - addresses:
      22	- ip: 192.168.169.184
      23	- ip: 192.168.169.165
      24	- ip: 39.156.69.79
1. ports:
2. - name: abc  ## ep和service要是一样的
3. port: 80
4. protocol: TCP
   场景：Pod要访问 MySQL。 MySQL单独部署到很多机器，每次记ip麻烦
### 集群内创建一个Service，实时的可以剔除EP信息。反向代理集群外的东西。

2、ClusterIP





手动指定的ClusterIP必须在合法范围内
1. type: ClusterIP
2. ClusterIP: 手动指定/None/""
   None会创建出没有ClusterIP的headless  service（无头服务），Pod需要用服务的域名访问


3、NodePort



1. apiVersion: v1
2. kind: Service
3. metadata:
4. name: my-service
5. namespace: default
6. type: NodePort
7. ports:
8. - protocol: TCP
9. port: 80  # service 80
10. targetPort: 80  #目标80
11. nodePort: 32123  #自定义



如果将
type
--service-node-port-range
字段设置为 NodePort ，则 Kubernetes 将在
标志指
定的范围内分配端口（默认值：30000-32767）
k8s集群的所有机器都将打开监听这个端口的数据，访问任何一个机器，都可以访问这个service对应的Pod
使用 nodePort 自定义端口

\
4、ExternalName












其他的Pod可以通过访问这个service而访问其他的域名服务但是需要注意目标服务的跨域问题
1. apiVersion: v1
2. kind: Service
3. metadata:
4. name: my-service-05
5. namespace: default
6. spec:
7. type: ExternalName
8. externalName: baidu.com

5、LoadBalancer
1. apiVersion: v1
2. kind: Service
3. metadata:
4. creationTimestamp: null
5. labels:
6. app.kubernetes.io/name:  load-balancer-example
7. name: my-service
8. spec:
9. ports:
10. - port: 80
11. protocol: TCP
12. targetPort:  80



13
14
15
selector:
app.kubernetes.io/name: load-balancer-example type: LoadBalancer

6、扩展 - externalIP

在 Service 的定义中， externalIPs 可以和任何类型的 .spec.type 一通使用。在下面的例子中，客户端可通过 80.11.12.10:80 （externalIP:port） 访问 my-service

1. apiVersion: v1
2. kind: Service
3. metadata:
4. name: my-service-externalip
5. spec:
6. selector:
7. app: canary-nginx
8. ports:
9. - name: http
10. protocol: TCP
11. port: 80
12. targetPort:  80
13. externalIPs: ### 定义只有externalIPs指定的地址才可以访问这个service
    14	- 10.170.0.111

7、扩展 - Pod的DNS

1. apiVersion: v1
2. kind: Service
3. metadata:
4. name: default-subdomain
5. spec:
6. selector:
7. name: busybox
8. clusterIP: None
9. ports:
10. - name: foo # 实际上不需要指定端口号
      11	port: 1234
1. targetPort:  1234
2. ---
3. apiVersion: v1
4. kind: Pod
5. metadata:
6. name: busybox1
7. labels:
8. name: busybox
9. spec:
10. hostname: busybox-1
11. subdomain: default-subdomain































访问 busybox-1. default-subdomain .default. svc.cluster.local 可以访问到busybox-1。访问Service
1. 	## 指定必须和svc名称一样，才可以 podName.subdomain.名称空间.svc.cluster.local访问。否则访问不同指定Pod
2. containers:
3. - image: busybox:1.28
4. command:
5. - sleep
6. - "3600"
7. name: busybox
8. ---
9. apiVersion: v1
10. kind: Pod
11. metadata:
12. name: busybox2
13. labels:
14. name: busybox
15. spec:
16. hostname: busybox-2
17. subdomain: default-subdomain
18. containers:
19. - image: busybox:1.28
20. command:
21. - sleep
22. - "3600"
23. name: busybox
    同名称空间
    ping service-name 即可  不同名称空间
    ping service-name.namespace 即可
    访问Pod
    同名称空间
    ping pod-host-name.service-name 即可  不同名称空间
    ping pod-host-name.service-name.namespace 即可








三、Ingress
为什么需要Ingress？
Service可以使用NodePort暴露集群外访问端口，但是性能低下不安全  缺少Layer7的统一访问入口，可以负载均衡、限流等
Ingress 公开了从集群外部到集群内服务的 HTTP 和 HTTPS 路由。 流量路由由 Ingress 资源上定义的规则控制。
我们使用Ingress作为整个集群统一的入口，配置Ingress规则转到对应的Service



1、Ingress nginx和nginx ingress

1、nginx ingress
这是nginx官方做的，适配k8s的，分为开源版和nginx plus版（收费）。文档地址
https://www.nginx.com/products/nginx-ingress-controller




2、ingress nginx

https://kubernetes.io/zh/docs/concepts/services-networking/ingress/#ingress-
%E6%98%AF%E4%BB%80%E4%B9%88
这是k8s官方做的，适配nginx的。这个里面会及时更新一些特性，而且性能很高，也被广泛采用。文档地址

1. ##  默认安装使用这个镜像
2. registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/ingress-nginx-controller:v0.46.0


2、ingress nginx 安装

1、安装
自建集群使用 裸金属安装方式
需要如下修改：
修改ingress-nginx-controller镜像为 registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/ingress-nginx- controller:v0.46.0
修改Deployment为DaemonSet比较好
修改Container使用主机网络，直接在主机上开辟 80,443端口，无需中间解析，速度更快  Container使用主机网络，对应的dnsPolicy策略也需要改为主机网络的
修改Service为ClusterIP，无需NodePort模式了
修改DaemonSet的nodeSelector:	。这样只需要给node节点打上 ingress-
ingress-node=true
标签，即可快速的加入/剔除 ingress-controller的数量
node=true

修改好的yaml如下。大家直接复制使用

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
apiVersion: v1 kind: Namespace metadata:
name: ingress-nginx labels:
app.kubernetes.io/name:  ingress-nginx
app.kubernetes.io/instance:  ingress-nginx
---
# Source: ingress-nginx/templates/controller-serviceaccount.yaml apiVersion: v1
kind: ServiceAccount metadata:
labels:
helm.sh/chart: ingress-nginx-3.30.0 app.kubernetes.io/name: ingress-nginx app.kubernetes.io/instance: ingress-nginx app.kubernetes.io/version: 0.46.0 app.kubernetes.io/managed-by: Helm app.kubernetes.io/component: controller
name: ingress-nginx

1. namespace: ingress-nginx
2. automountServiceAccountToken: true
3. ---
4. # Source: ingress-nginx/templates/controller-configmap.yaml
5. apiVersion: v1
6. kind: ConfigMap
7. metadata:
8. labels:
9. helm.sh/chart:  ingress-nginx-3.30.0
10. app.kubernetes.io/name:  ingress-nginx
11. app.kubernetes.io/instance:  ingress-nginx
12. app.kubernetes.io/version: 0.46.0
13. app.kubernetes.io/managed-by: Helm
14. app.kubernetes.io/component: controller
15. name: ingress-nginx-controller
16. namespace: ingress-nginx
17. data:
18. ---
19. # Source: ingress-nginx/templates/clusterrole.yaml
20. apiVersion: rbac.authorization.k8s.io/v1
21. kind: ClusterRole
22. metadata:
23. labels:
24. helm.sh/chart:  ingress-nginx-3.30.0
25. app.kubernetes.io/name:  ingress-nginx
26. app.kubernetes.io/instance:  ingress-nginx
27. app.kubernetes.io/version: 0.46.0
28. app.kubernetes.io/managed-by: Helm
29. name: ingress-nginx
30. rules:
31. - apiGroups:
      54	- ''
1. resources:
2. - configmaps
3. - endpoints
4. - nodes
5. - pods
6. - secrets
7. verbs:
8. - list
9. - watch
10. - apiGroups:
      65	- ''
1. resources:
2. - nodes
3. verbs:
4. - get
5. - apiGroups:
     71	- ''
1. resources:
2. - services
3. verbs:
4. - get
5. - list
6. - watch
7. - apiGroups:
8. - extensions
9. - networking.k8s.io	# k8s 1.14+
10. resources:
11. - ingresses
12. verbs:
13. - get
14. - list
15. - watch
16. - apiGroups:
      88	- ''
1. resources:
2. - events
3. verbs:
4. - create
5. - patch
6. - apiGroups:
7. - extensions
8. - networking.k8s.io	# k8s 1.14+
9. resources:
10. - ingresses/status
11. verbs:
12. - update
13. - apiGroups:
14. - networking.k8s.io	# k8s 1.14+
15. resources:
16. - ingressclasses
17. verbs:
18. - get
19. - list
20. - watch
21. ---
22. # Source: ingress-nginx/templates/clusterrolebinding.yaml
23. apiVersion: rbac.authorization.k8s.io/v1
24. kind: ClusterRoleBinding
25. metadata:
26. labels:
27. helm.sh/chart:  ingress-nginx-3.30.0
28. app.kubernetes.io/name:  ingress-nginx
29. app.kubernetes.io/instance:  ingress-nginx
30. app.kubernetes.io/version: 0.46.0
31. app.kubernetes.io/managed-by: Helm
32. name: ingress-nginx
33. roleRef:
34. apiGroup: rbac.authorization.k8s.io
35. kind: ClusterRole
36. name: ingress-nginx
37. subjects:
38. - kind: ServiceAccount
39. name: ingress-nginx
40. namespace: ingress-nginx
41. ---
42. # Source: ingress-nginx/templates/controller-role.yaml
43. apiVersion: rbac.authorization.k8s.io/v1
44. kind: Role
45. metadata:
46. labels:
47. helm.sh/chart:  ingress-nginx-3.30.0
48. app.kubernetes.io/name:  ingress-nginx
49. app.kubernetes.io/instance:  ingress-nginx
50. app.kubernetes.io/version: 0.46.0
51. app.kubernetes.io/managed-by: Helm
52. app.kubernetes.io/component: controller
    141	name: ingress-nginx
    142	namespace: ingress-nginx
    143	rules:
    144	- apiGroups:
    145	- ''
    146	resources:
    147	- namespaces
    148	verbs:
    149	- get
    150	- apiGroups:
    151	- ''
    152	resources:
    153	- configmaps
    154	- pods
    155	- secrets
    156	- endpoints
    157	verbs:
    158	- get
    159	- list
    160	- watch
    161	- apiGroups:
    162	- ''
    163	resources:
    164	- services
    165	verbs:
    166	- get
    167	- list
    168	- watch
    169	- apiGroups:
    170	- extensions
    171	- networking.k8s.io	# k8s 1.14+
    172	resources:
    173	- ingresses
    174	verbs:
    175	- get
    176	- list
    177	- watch
    178	- apiGroups:
    179	- extensions
    180	- networking.k8s.io	# k8s 1.14+
    181	resources:
    182	- ingresses/status
    183	verbs:
    184	- update
    185	- apiGroups:
    186	- networking.k8s.io	# k8s 1.14+
    187	resources:
    188	- ingressclasses
    189	verbs:
    190	- get
    191	- list
    192	- watch
    193	-	apiGroups:
    194		- ''
    195		resources:
    196		- configmaps

1. resourceNames:
2. - ingress-controller-leader-nginx
3. verbs:
4. - get
5. - update
6. - apiGroups:
     203	- ''
1. resources:
2. - configmaps
3. verbs:
4. - create
5. - apiGroups:
     209	- ''
1. resources:
2. - events
3. verbs:
4. - create
5. - patch
6. ---
7. # Source: ingress-nginx/templates/controller-rolebinding.yaml
8. apiVersion: rbac.authorization.k8s.io/v1
9. kind: RoleBinding
10. metadata:
11. labels:
12. helm.sh/chart:  ingress-nginx-3.30.0
13. app.kubernetes.io/name:  ingress-nginx
14. app.kubernetes.io/instance:  ingress-nginx
15. app.kubernetes.io/version: 0.46.0
16. app.kubernetes.io/managed-by: Helm
17. app.kubernetes.io/component: controller
18. name: ingress-nginx
19. namespace: ingress-nginx
20. roleRef:
21. apiGroup: rbac.authorization.k8s.io
22. kind: Role
23. name: ingress-nginx
24. subjects:
25. - kind: ServiceAccount
26. name: ingress-nginx
27. namespace: ingress-nginx
28. ---
29. # Source: ingress-nginx/templates/controller-service-webhook.yaml
30. apiVersion: v1
31. kind: Service
32. metadata:
33. labels:
34. helm.sh/chart:  ingress-nginx-3.30.0
35. app.kubernetes.io/name:  ingress-nginx
36. app.kubernetes.io/instance:  ingress-nginx
37. app.kubernetes.io/version: 0.46.0
38. app.kubernetes.io/managed-by: Helm
39. app.kubernetes.io/component: controller
40. name: ingress-nginx-controller-admission
41. namespace: ingress-nginx
42. spec:
43. type: ClusterIP
44. ports:
45. - name: https-webhook
46. port: 443
47. targetPort:  webhook
48. selector:
49. app.kubernetes.io/name:  ingress-nginx
50. app.kubernetes.io/instance:  ingress-nginx
51. app.kubernetes.io/component: controller
52. ---
53. # Source: ingress-nginx/templates/controller-service.yaml：不要
54. apiVersion: v1
55. kind: Service
56. metadata:
57. annotations:
58. labels:
59. helm.sh/chart:  ingress-nginx-3.30.0
60. app.kubernetes.io/name:  ingress-nginx
61. app.kubernetes.io/instance:  ingress-nginx
62. app.kubernetes.io/version: 0.46.0
63. app.kubernetes.io/managed-by: Helm
64. app.kubernetes.io/component: controller
65. name: ingress-nginx-controller
66. namespace: ingress-nginx
67. spec:
68. type: ClusterIP ## 改为clusterIP
69. ports:
70. - name: http
71. port: 80
72. protocol: TCP
73. targetPort: http
74. - name: https
75. port: 443
76. protocol: TCP
77. targetPort: https
78. selector:
79. app.kubernetes.io/name:  ingress-nginx
80. app.kubernetes.io/instance:  ingress-nginx
81. app.kubernetes.io/component: controller
82. ---
83. # Source: ingress-nginx/templates/controller-deployment.yaml
84. apiVersion: apps/v1
85. kind: DaemonSet
86. metadata:
87. labels:
88. helm.sh/chart:  ingress-nginx-3.30.0
89. app.kubernetes.io/name:  ingress-nginx
90. app.kubernetes.io/instance:  ingress-nginx
91. app.kubernetes.io/version: 0.46.0
92. app.kubernetes.io/managed-by: Helm
93. app.kubernetes.io/component: controller
94. name: ingress-nginx-controller
95. namespace: ingress-nginx
96. spec:
97. selector:
98. matchLabels:
99. app.kubernetes.io/name:  ingress-nginx
100. app.kubernetes.io/instance:  ingress-nginx
101. app.kubernetes.io/component: controller
102. revisionHistoryLimit: 10
103. minReadySeconds: 0
104. template:
105. metadata:
106. labels:
107. app.kubernetes.io/name:  ingress-nginx
108. app.kubernetes.io/instance:  ingress-nginx
109. app.kubernetes.io/component: controller
110. spec:
111. dnsPolicy: ClusterFirstWithHostNet	## dns对应调整为主机网络
112. hostNetwork: true ## 直接让nginx占用本机80端口和443端口，所以使用主机网络
113. containers:
114. - name: controller
115. 	image: registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/ingress- nginx-controller:v0.46.0
116. imagePullPolicy: IfNotPresent
117. lifecycle:
118. preStop:
119. exec:
120. command:
121. - /wait-shutdown
122. args:
123. - /nginx-ingress-controller
124. - --election-id=ingress-controller-leader
125. - --ingress-class=nginx
126. -   --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
127. - --validating-webhook=:8443
128. -   --validating-webhook-certificate=/usr/local/certificates/cert
129. - --validating-webhook-key=/usr/local/certificates/key
130. securityContext:
131. capabilities:
132. drop:
133. - ALL
134. add:
135. - NET_BIND_SERVICE
136. runAsUser: 101
137. allowPrivilegeEscalation: true
138. env:
139. - name: POD_NAME
140. valueFrom:
141. fieldRef:
142. fieldPath: metadata.name
143. - name: POD_NAMESPACE
144. valueFrom:
145. fieldRef:
146. fieldPath: metadata.namespace
147. - name: LD_PRELOAD
148. value: /usr/local/lib/libmimalloc.so
149. livenessProbe:
150. httpGet:
151. path: /healthz
152. port: 10254
153. scheme: HTTP
154. initialDelaySeconds: 10
155. periodSeconds: 10
156. timeoutSeconds: 1
157. successThreshold: 1
158. failureThreshold: 5
159. readinessProbe:
160. httpGet:
161. path: /healthz
162. port: 10254
163. scheme: HTTP
164. initialDelaySeconds: 10
165. periodSeconds: 10
166. timeoutSeconds: 1
167. successThreshold: 1
168. failureThreshold: 3
169. ports:
170. - name: http
171. containerPort: 80
172. protocol: TCP
173. - name: https
174. containerPort: 443
175. protocol: TCP
176. - name: webhook
177. containerPort:  8443
178. protocol: TCP
179. volumeMounts:
180. - name: webhook-cert
181. mountPath: /usr/local/certificates/
182. readOnly: true
183. resources:
184. requests:
185. cpu: 100m
186. memory: 90Mi
187. nodeSelector:
188. 	node-role: ingress #以后只需要给某个node打上这个标签就可以部署ingress-nginx到这个节点上了
189. #kubernetes.io/os: linux  ## 修改节点选择
190. serviceAccountName:  ingress-nginx
191. terminationGracePeriodSeconds: 300
192. volumes:
193. - name: webhook-cert
194. secret:
195. secretName: ingress-nginx-admission
196. ---
197. #  Source:  ingress-nginx/templates/admission-webhooks/validating-webhook.yaml
198. # before changing this value, check the required kubernetes version
199. # https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission- controllers/#prerequisites
200. apiVersion: admissionregistration.k8s.io/v1
201. kind: ValidatingWebhookConfiguration
202. metadata:
203. labels:
204. helm.sh/chart:  ingress-nginx-3.30.0
205. app.kubernetes.io/name:  ingress-nginx
206. app.kubernetes.io/instance:  ingress-nginx
207. app.kubernetes.io/version: 0.46.0
208. app.kubernetes.io/managed-by: Helm
209. app.kubernetes.io/component: admission-webhook
210. name: ingress-nginx-admission
211. webhooks:
212. - name: validate.nginx.ingress.kubernetes.io
213. matchPolicy: Equivalent
214. rules:
215. - apiGroups:
216. - networking.k8s.io
       426
       427
       428
       429
       430
       431
       432
       433
       434
       435
       436
       437
       438
       439
       440
       441
       442
       443






















---
apiVersions:
● v1beta1 operations:
● CREATE
● UPDATE
resources:
● ingresses failurePolicy: Fail sideEffects: None admissionReviewVersions:
● v1
● v1beta1 clientConfig:
service:
namespace: ingress-nginx
name: ingress-nginx-controller-admission path: /networking/v1beta1/ingresses

1. # Source: ingress-nginx/templates/admission-webhooks/job- patch/serviceaccount.yaml
2. apiVersion: v1
3. kind: ServiceAccount
4. metadata:
5. name: ingress-nginx-admission
6. annotations:
7. helm.sh/hook:  pre-install,pre-upgrade,post-install,post-upgrade
8. helm.sh/hook-delete-policy:  before-hook-creation,hook-succeeded
9. labels:
10. helm.sh/chart:  ingress-nginx-3.30.0
11. app.kubernetes.io/name:  ingress-nginx
12. app.kubernetes.io/instance:  ingress-nginx
13. app.kubernetes.io/version: 0.46.0
14. app.kubernetes.io/managed-by: Helm
15. app.kubernetes.io/component: admission-webhook
16. namespace: ingress-nginx
17. ---
18. # Source: ingress-nginx/templates/admission-webhooks/job-patch/clusterrole.yaml
19. apiVersion: rbac.authorization.k8s.io/v1
20. kind: ClusterRole
21. metadata:
22. name: ingress-nginx-admission
23. annotations:
24. helm.sh/hook:  pre-install,pre-upgrade,post-install,post-upgrade
25. helm.sh/hook-delete-policy:  before-hook-creation,hook-succeeded
26. labels:
27. helm.sh/chart:  ingress-nginx-3.30.0
28. app.kubernetes.io/name:  ingress-nginx
29. app.kubernetes.io/instance:  ingress-nginx
30. app.kubernetes.io/version: 0.46.0
31. app.kubernetes.io/managed-by: Helm
32. app.kubernetes.io/component: admission-webhook
33. rules:
34. - apiGroups:
35. - admissionregistration.k8s.io
36. resources:
37. - validatingwebhookconfigurations
38. verbs:
39. - get
      483
      484

---
● update
1. # Source: ingress-nginx/templates/admission-webhooks/job- patch/clusterrolebinding.yaml
2. apiVersion: rbac.authorization.k8s.io/v1
3. kind: ClusterRoleBinding
4. metadata:
5. name: ingress-nginx-admission
6. annotations:
7. helm.sh/hook:  pre-install,pre-upgrade,post-install,post-upgrade
8. helm.sh/hook-delete-policy:  before-hook-creation,hook-succeeded
9. labels:
10. helm.sh/chart:  ingress-nginx-3.30.0
11. app.kubernetes.io/name:  ingress-nginx
12. app.kubernetes.io/instance:  ingress-nginx
13. app.kubernetes.io/version: 0.46.0
14. app.kubernetes.io/managed-by: Helm
15. app.kubernetes.io/component: admission-webhook
16. roleRef:
17. apiGroup: rbac.authorization.k8s.io
18. kind: ClusterRole
19. name: ingress-nginx-admission
20. subjects:
21. - kind: ServiceAccount
22. name: ingress-nginx-admission
23. namespace: ingress-nginx
24. ---
25. # Source: ingress-nginx/templates/admission-webhooks/job-patch/role.yaml
26. apiVersion: rbac.authorization.k8s.io/v1
27. kind: Role
28. metadata:
29. name: ingress-nginx-admission
30. annotations:
31. helm.sh/hook:  pre-install,pre-upgrade,post-install,post-upgrade
32. helm.sh/hook-delete-policy:  before-hook-creation,hook-succeeded
33. labels:
34. helm.sh/chart:  ingress-nginx-3.30.0
35. app.kubernetes.io/name:  ingress-nginx
36. app.kubernetes.io/instance:  ingress-nginx
37. app.kubernetes.io/version: 0.46.0
38. app.kubernetes.io/managed-by: Helm
39. app.kubernetes.io/component: admission-webhook
40. namespace: ingress-nginx
41. rules:
42. - apiGroups:
      527	- ''
1. resources:
2. - secrets
3. verbs:
4. - get
5. - create
6. ---
7. # Source: ingress-nginx/templates/admission-webhooks/job-patch/rolebinding.yaml
8. apiVersion: rbac.authorization.k8s.io/v1
9. kind: RoleBinding
10. metadata:
11. name: ingress-nginx-admission
12. annotations:
13. helm.sh/hook:  pre-install,pre-upgrade,post-install,post-upgrade
14. helm.sh/hook-delete-policy:  before-hook-creation,hook-succeeded
15. labels:
16. helm.sh/chart:  ingress-nginx-3.30.0
17. app.kubernetes.io/name:  ingress-nginx
18. app.kubernetes.io/instance:  ingress-nginx
19. app.kubernetes.io/version: 0.46.0
20. app.kubernetes.io/managed-by: Helm
21. app.kubernetes.io/component: admission-webhook
22. namespace: ingress-nginx
23. roleRef:
24. apiGroup: rbac.authorization.k8s.io
25. kind: Role
26. name: ingress-nginx-admission
27. subjects:
28. - kind: ServiceAccount
29. name: ingress-nginx-admission
30. namespace: ingress-nginx
31. ---
32. # Source: ingress-nginx/templates/admission-webhooks/job-patch/job- createSecret.yaml
33. apiVersion: batch/v1
34. kind: Job
35. metadata:
36. name: ingress-nginx-admission-create
37. annotations:
38. helm.sh/hook:  pre-install,pre-upgrade
39. helm.sh/hook-delete-policy:  before-hook-creation,hook-succeeded
40. labels:
41. helm.sh/chart:  ingress-nginx-3.30.0
42. app.kubernetes.io/name:  ingress-nginx
43. app.kubernetes.io/instance:  ingress-nginx
44. app.kubernetes.io/version: 0.46.0
45. app.kubernetes.io/managed-by: Helm
46. app.kubernetes.io/component: admission-webhook
47. namespace: ingress-nginx
48. spec:
49. template:
50. metadata:
51. name: ingress-nginx-admission-create
52. labels:
53. helm.sh/chart:  ingress-nginx-3.30.0
54. app.kubernetes.io/name:  ingress-nginx
55. app.kubernetes.io/instance:  ingress-nginx
56. app.kubernetes.io/version: 0.46.0
57. app.kubernetes.io/managed-by: Helm
58. app.kubernetes.io/component: admission-webhook
59. spec:
60. containers:
61. - name: create
62. image:  docker.io/jettech/kube-webhook-certgen:v1.5.1
63. imagePullPolicy: IfNotPresent
64. args:
65. - create
66. 	- --host=ingress-nginx-controller-admission,ingress-nginx- controller-admission.$(POD_NAMESPACE).svc
67. - --namespace=$(POD_NAMESPACE)
68. - --secret-name=ingress-nginx-admission
      596
      597
      598
      599
      600
      601
      602
      603
      604
      605
      606













---
env:
- name: POD_NAMESPACE
  valueFrom:
  fieldRef:
  fieldPath: metadata.namespace restartPolicy: OnFailure serviceAccountName: ingress-nginx-admission securityContext:
  runAsNonRoot: true runAsUser: 2000

1. # Source: ingress-nginx/templates/admission-webhooks/job-patch/job- patchWebhook.yaml
2. apiVersion: batch/v1
3. kind: Job
4. metadata:
5. name: ingress-nginx-admission-patch
6. annotations:
7. helm.sh/hook: post-install,post-upgrade
8. helm.sh/hook-delete-policy:  before-hook-creation,hook-succeeded
9. labels:
10. helm.sh/chart:  ingress-nginx-3.30.0
11. app.kubernetes.io/name:  ingress-nginx
12. app.kubernetes.io/instance:  ingress-nginx
13. app.kubernetes.io/version: 0.46.0
14. app.kubernetes.io/managed-by: Helm
15. app.kubernetes.io/component: admission-webhook
16. namespace: ingress-nginx
17. spec:
18. template:
19. metadata:
20. name: ingress-nginx-admission-patch
21. labels:
22. helm.sh/chart:  ingress-nginx-3.30.0
23. app.kubernetes.io/name:  ingress-nginx
24. app.kubernetes.io/instance:  ingress-nginx
25. app.kubernetes.io/version: 0.46.0
26. app.kubernetes.io/managed-by: Helm
27. app.kubernetes.io/component: admission-webhook
28. spec:
29. containers:
30. - name: patch
31. image:  docker.io/jettech/kube-webhook-certgen:v1.5.1
32. imagePullPolicy: IfNotPresent
33. args:
34. - patch
35. - --webhook-name=ingress-nginx-admission
36. - --namespace=$(POD_NAMESPACE)
37. - --patch-mutating=false
38. - --secret-name=ingress-nginx-admission
39. - --patch-failure-policy=Fail
40. env:
41. - name: POD_NAMESPACE
42. valueFrom:
43. fieldRef:
44. fieldPath: metadata.namespace
45. restartPolicy: OnFailure
46. serviceAccountName: ingress-nginx-admission

653
654
655
securityContext:
runAsNonRoot: true runAsUser: 2000

2、验证
访问部署了ingress-nginx主机的80端口，有nginx响应即可。

2、卸载
即可
kubectl delete -f ingress-controller.yaml

3、案例实战
1、基本配置

1. apiVersion: networking.k8s.io/v1
2. kind: Ingress
3. metadata:
4. name: itdachang-ingress
5. namespace: default
6. spec:
7. rules:
8. - host: itdachang.com
9. http:
10. paths:
11. - path: /
12. pathType: Prefix
13. backend:  ## 指定需要响应的后端服务
14. service:
15. name: my-nginx-svc  ## kubernetes集群的svc名称
16. port:
17. number: 80  ## service的端口号


pathType 详细：
Prefix ：基于以

分隔的 URL 路径前缀匹配。匹配区分大小写，并且对路径中的元素逐个

/
完成。 路径元素指的是由	分隔符分隔的路径中的标签列表。 如果每个 p 都是请求路径

/
p 的元素前缀，则请求与路径 p 匹配。
Exact ：精确匹配 URL 路径，且区分大小写。
ImplementationSpecific  ：对于这种路径类型，匹配方法取决于 IngressClass。 具体实现
Prefix
Exact
可以将其作为单独的 pathType 处理或者与
或
类型作相同处理。

2、默认后端

1. apiVersion: networking.k8s.io/v1
2. kind: Ingress


1. metadata:
2. name: itdachang-ingress
3. namespace: default
4. spec:
5. defaultBackend:   ##  指定所有未匹配的默认后端
6. service:
7. name: php-apache
8. port:
9. number: 80
10. rules:
11. - host: itdachang.com
12. http:
13. paths:
14. - path: /abc
15. pathType: Prefix
16. backend:
17. service:
18. name: my-nginx-svc
19. port:
20. number: 80

效果
itdachang.com 下的 非 /abc 开头的所有请求，都会到defaultBackend非itdachang.com 域名下的所有请求，也会到defaultBackend

3、路径重写
https://kubernetes.github.io/ingress-nginx/examples/rewrite/

Rewrite 功能，经常被用于前后分离的场景
前端给服务器发送 / 请求映射前端地址。
后端给服务器发送 /api 请求来到对应的服务。但是后端服务没有 /api的起始路径，所以需要 ingress-controller自动截串


1. apiVersion: networking.k8s.io/v1
2. kind: Ingress
3. metadata:
4. annotations:
5. nginx.ingress.kubernetes.io/rewrite-target: /$2
6. name: rewrite
7. namespace: default
8. spec:
9. rules:
10. - host: itdachang.com
11. http:
12. paths:
13. - backend:
14. service:
15. name: php-apache
16. port:
17. number: 80


18
19
path: /api(/|$)(.*)
pathType: Prefix



4、配置SSL
https://kubernetes.github.io/ingress-nginx/user-guide/tls/
生成证书：（也可以去青云申请免费证书进行配置）

1	$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ${KEY_FILE:tls.key}
-out ${CERT_FILE:tls.cert} -subj "/CN=${HOST:itdachang.com}/O=${HOST:itdachang.com}"
2
3	kubectl create secret tls ${CERT_NAME:itdachang-tls} --key ${KEY_FILE:tls.key} -- cert ${CERT_FILE:tls.cert}
4
5
6	## 示例命令如下
7	openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.cert
-subj "/CN=itdachang.com/O=itdachang.com"
8
9	kubectl create secret tls itdachang-tls --key tls.key --cert tls.cert
配置域名使用证书；

1. apiVersion: networking.k8s.io/v1
2. kind: Ingress
3. metadata:
4. name: itdachang-ingress
5. namespace: default
6. spec:
7. tls:
8. - hosts:
9. - itdachang.com
10. secretName: itdachang-tls
11. rules:
12. - host: itdachang.com
13. http:
14. paths:
15. - path: /
16. pathType: Prefix
17. backend:
18. service:
19. name: my-nginx-svc
20. port:
21. number: 80
    配置好证书，访问域名，就会默认跳转到https；


5、限速
https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#rate-limiting
6、灰度发布-Canary
以前可以使用k8s的Service配合Deployment进行金丝雀部署。原理如下

缺点：
不能自定义灰度逻辑，比如指定用户进行灰度

现在可以使用Ingress进行灰度。原理如下

1. ##  使用如下文件部署两个service版本。v1版本返回nginx默认页，v2版本返回  11111
2. apiVersion: v1
3. kind: Service


1. metadata:
2. name: v1-service
3. namespace: default
4. spec:
5. selector:
6. app: v1-pod
7. type: ClusterIP
8. ports:
9. - name: http
10. port: 80
11. targetPort:  80
12. protocol: TCP
13. ---
14. apiVersion: apps/v1
15. kind: Deployment
16. metadata:
17. name: v1-deploy
18. namespace: default
19. labels:
20. app:  v1-deploy
21. spec:
22. selector:
23. matchLabels:
24. app: v1-pod
25. replicas: 1
26. template:
27. metadata:
28. labels:
29. app:  v1-pod
30. spec:
31. containers:
32. - name: nginx
33. image: nginx
34. ---
35. apiVersion: v1
36. kind: Service
37. metadata:
38. name: canary-v2-service
39. namespace: default
40. spec:
41. selector:
42. app: canary-v2-pod
43. type: ClusterIP
44. ports:
45. - name: http
46. port: 80
47. targetPort:  80
48. protocol: TCP
49. ---
50. apiVersion: apps/v1
51. kind: Deployment
52. metadata:
53. name:  canary-v2-deploy
54. namespace: default
55. labels:
56. app:  canary-v2-deploy
57. spec:
58. selector:

1. matchLabels:
2. app: canary-v2-pod
3. replicas: 1
4. template:
5. metadata:
6. labels:
7. app:  canary-v2-pod
8. spec:
9. containers:
10. - name: nginx
11. 	image: registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/nginx-test:env- msg



7、会话保持-Session亲和性
https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#session-affinity
第一次访问，ingress-nginx会返回给浏览器一个Cookie，以后浏览器带着这个Cookie，保证访问总是抵达之前的Pod；

1. ##   部署一个三个Pod的Deployment并设置Service
2. apiVersion: v1
3. kind: Service
4. metadata:
5. name:  session-affinity
6. namespace: default
7. spec:
8. selector:
9. app: session-affinity
10. type: ClusterIP
11. ports:
12. - name: session-affinity
13. port: 80
14. targetPort:  80
15. protocol: TCP
16. ---
17. apiVersion: apps/v1
18. kind: Deployment
19. metadata:
20. name:  session-affinity
21. namespace: default
22. labels:
23. app:  session-affinity
24. spec:
25. selector:
26. matchLabels:
27. app: session-affinity
28. replicas: 3
29. template:
30. metadata:
31. labels:
32. app:  session-affinity
33. spec:
34. containers:


35
36
- name:  session-affinity
  image: nginx

编写具有会话亲和的ingress

四、NetworkPolicy
https://kubernetes.io/zh/docs/concepts/services-networking/network-policies/
指定Pod间的网络隔离策略，默认是所有互通。
Pod 之间互通，是通过如下三个标识符的组合来辩识的：
a. 其他被允许的 Pods（例外：Pod 无法阻塞对自身的访问）
b. 被允许的名称空间
c. IP 组块（例外：与 Pod 运行所在的节点的通信总是被允许的， 无论 Pod 或节点的 IP 地址）


1、Pod隔离与非隔离
默认情况下，Pod 都是非隔离的（non-isolated），可以接受来自任何请求方的网络请求。
如果一个 NetworkPolicy 的标签选择器选中了某个 Pod，则该 Pod 将变成隔离的（isolated），并将拒绝任何不被 NetworkPolicy 许可的网络连接。
2、规约

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
apiVersion: networking.k8s.io/v1 kind: NetworkPolicy
metadata:
name: test-network-policy namespace: default
spec:
podSelector: ## 选中指定Pod matchLabels:
role: db
policyTypes:  ## 定义上面Pod的入站出站规则
● Ingress
● Egress
ingress:
- from:
## 定义入站白名单
- ipBlock:
















基本信息： 同其他的 Kubernetes 对象一样， NetworkPolicy 需要 apiVersion 、 kind 、字段
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
cidr: 172.17.0.0/16
except:
- 172.17.1.0/24
  ● namespaceSelector: matchLabels:
  project: myproject
  ● podSelector: matchLabels:
  role: frontend
  ports:
  ● protocol: TCP port: 6379
  egress:  ## 定义出站白名单
  ● to:
  ○ ipBlock:
  cidr: 10.0.0.0/24 ports:
  ○ protocol: TCP
  port: 5978
  metadata
  spec： NetworkPolicy 的spec字段包含了定义网络策略的主要信息：
  podSelector： 同名称空间中，符合此标签选择器 .spec.podSelector 的 Pod 都将应用这个 NetworkPolicy 。上面的 Example中的 podSelector 选择了 role=db 的 Pod。如果该字段为空，则将对名称空间中所有的 Pod 应用这个 NetworkPolicy
  policyTypes： .spec.policyTypes 是一个数组类型的字段，该数组中可以包含
  NetworkPolicy
  Ingress 、 Egress  中的一个，也可能两个都包含。该字段标识了此
  否应用到  入方向的网络流量、出方向的网络流量、或者两者都有。如果不指定
  NetworkPolicy
  policyTypes 字段，该字段默认将始终包含 Ingress ，当向的规则时， Egress 也将被添加到默认值。
  是
  中包含出方
  ingress：ingress是一个数组，代表入方向的白名单规则。每一条规则都将允许与 from 和 ports 匹配的入方向的网络流量发生。例子中的 ingress 包含了一条规则，允许的入方向网络流量必须符合如下条件：
  Pod 的监听端口为
  6379
  请求方可以是如下三种来源当中的任意一种：
  ipBlock 为	网段，但是不包括 172.17.1.0/24 网段
  172.17.0.0/16
  namespaceSelector 标签选择器，匹配标签为  project=myproject
  podSelector 标签选择器，匹配标签为 role=frontend
  egress： egress 是一个数组，代表出方向的白名单规则。每一条规则都将允许与 to 和
  egress
  ports 匹配的出方向的网络流量发生。例子中的下条件：
  目标端口为 5978
  目标 ipBlock 为 10.0.0.0/24 网段
  允许的出方向网络流量必须符合如
  因此，例子中的	对网络流量做了如下限制：
  NetworkPolicy
  default
  role=db
1. 隔离了量
   名称空间中带有
   标签的所有 Pod 的入方向网络流量和出方向网络流
1. Ingress规则（入方向白名单规则）：
   当请求方是如下三种来源当中的任意一种时，允许访问  default  名称空间中所有带
   role=db 标签的 Pod 的6379端口：
   ipBlock 为	网段，但是不包括 172.17.1.0/24 网段
   172.17.0.0/16
   namespaceSelector 标签选择器，匹配标签为   project=myproject
   podSelector 标签选择器，匹配标签为  role=frontend
1. Egress规则（出方向白名单规则）：
   当如下条件满足时，允许出方向的网络流量：
   目标端口为 5978
   目标 ipBlock 为 10.0.0.0/24 网段

3、to和from选择器的行为
.spec.egress.to


NetworkPolicy 的择器：
.spec.ingress.from
和
字段中，可以指定 4 种类型的标签选
podSelector 选择与
NetworkPolicy
向访问控制规则的目标
同名称空间中的 Pod 作为入方向访问控制规则的源或者出方
namespaceSelector 选择某个名称空间（其中所有的Pod）作为入方向访问控制规则的源或者出方向访问控制规则的目标

to
from
namespaceSelector
namespaceSelector 和 podSelector 在一个
/
条目中同时包含	和
将选中指定名称空间中的指定 Pod。此时请特别留意 YAML 的写法，如下所示：
podSelector

1
2
3
4
5
6
7
8
9
10
...
ingress:
- from:
- namespaceSelector: matchLabels:
  user: alice podSelector:
  matchLabels: role: client
  ...
  该例子中，podSelector 前面没有	减号，namespaceSelector 和 podSelector 是同一个 from 元素的两个
  字段，将选中带

-
user=alice
role=client
标签的名称空间中所有带
标签的 Pod。但是，下面的这
个 NetworkPolicy 含义是不一样的：

1
2
3
4
5
6
7
8
9
10
...
ingress:
● from:
○ namespaceSelector: matchLabels:
user: alice
○ podSelector: matchLabels:
role: client
...
后者，podSelector 前面带	减号，说明 namespaceSelector 和 podSelector 是 from 数组中的两个元
素，他们将选中 NetworkPolicy 同名称空间中带签的名称空间的所有 Pod。

-
role=client
user=alice
标签的对象，以及带	标

前者是交集关系，后者是并集关系
ipBlock 可选择 IP CIDR 范围作为入方向访问控制规则的源或者出方向访问控制规则的目标。这里应该指定的是集群外部的 IP，因为集群内部 Pod 的 IP 地址是临时分配的，且不可预测。
集群的入方向和出方向网络机制通常需要重写网络报文的 source 或者 destination IP。kubernetes 并未定
义应该在处理	之前还是之后再修改 source / destination IP，因此，在不同的云供应
NetworkPolicy
商、使用不同的网络插件时，最终的行为都可能不一样。这意味着：
对于入方向的网络流量，某些情况下，你可以基于实际的源 IP 地址过滤流入的报文；在另外一些情况下，NetworkPolicy 所处理的 "source IP" 可能是 LoadBalancer 的 IP 地址，或者其他地址
对于出方向的网络流量，基于 ipBlock 的策略可能有效，也可能无效

4、场景
https://kubernetes.io/zh/docs/concepts/services-networking/network-policies/#default-policies