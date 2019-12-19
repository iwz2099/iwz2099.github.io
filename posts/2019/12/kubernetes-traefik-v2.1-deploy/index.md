# kubernetes集群部署traefik2.1


#### 架构&概念

![traefik v2.1 router](/images/2019/12/routers.png) 

Traefik2.x版本相比1.7.x架构有很大变化，正如上边这张架构图，最主要的功能是支持了TCP协议、增加了Router概念。

这里我们采用在kubernetes集群部署Traefik2.1，业务访问通过haproxy请求到traefik Ingress，下边是搭建过程涉及到一些概念：

* EntryPoints：Traefik的网络入口，定义请求接受的端口(不分http或tcp)
* CRD：Kubernetes API的扩展
* IngressRouter：将传入请求转发到可以处理请求的服务，另外转发请求之前可以通过Middlewares动态更新请求
* Middlewares：请求到达服务之前进行动态处理请求参数，比如header或转发规则等等。
* TraefikService：如果果CRD定义了了这种类型，IngressRouter可以直接引用，处在IngressRouter和服务之间,类似于Maesh架构，更适合较为复杂场景

#### kubernetes配置SSL证书

因为业务服务使用https,这里先配置下SSL证书:
```
kubectl -n cattle-system create secret tls tls-rancher-ingress --cert=test.cn.pem  --key=test.cn.key
```

#### CRD配置
这里定义了IngressRoute、Middleware、TLSOption、IngressRouteTCP、TraefikService,其中TraefikService是在2.1版本新增加的CRD

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutes.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRoute
    plural: ingressroutes
    singular: ingressroute
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: middlewares.traefik.containo.us
spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: Middleware
    plural: middlewares
    singular: middleware
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: tlsoptions.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TLSOption
    plural: tlsoptions
    singular: tlsoption
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ingressroutetcps.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: IngressRouteTCP
    plural: ingressroutetcps
    singular: ingressroutetcp
  scope: Namespaced

---
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: traefikservices.traefik.containo.us

spec:
  group: traefik.containo.us
  version: v1alpha1
  names:
    kind: TraefikService
    plural: traefikservices
    singular: traefikservice
  scope: Namespaced
```
#### kubernetes配置RBAC

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller

rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - traefik.containo.us
    resources:
      - middlewares
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - traefik.containo.us
    resources:
      - ingressroutes
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - traefik.containo.us
    resources:
      - ingressroutetcps
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - traefik.containo.us
    resources:
      - tlsoptions
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - traefik.containo.us
    resources:
      - traefikservices
    verbs:
      - get
      - list
      - watch

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
  - kind: ServiceAccount
    name: traefik-ingress-controller
    namespace: kube-system
```

#### TLS参数配置
默认配置TLS1.2,当然也可以使用TLS1.3

```
apiVersion: traefik.containo.us/v1alpha1
kind: TLSOption
metadata:
  name: mytlsoption
  namespace: kube-system

spec:
  minversion: VersionTLS12
  snistrict: true
  ciphersuites:
    - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
    - TLS_RSA_WITH_AES_256_GCM_SHA384
    - TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
    - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
    - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
    - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
```

#### TraefikService配置
TraefikService有点类似Maesh解决服务之间的调用逻辑，只是Maesh依赖coredns；另外traefik service还可以设置后端服务权重，配置服务的流量镜像。

这里我们配置traefik dashboard和rancher的traefikservice类型服务，其他服务配置可以参考这里rancher的traefik service, traefik service会转发请求到kubernetes 的服务类型(上一章节我们已经通过helm3创建了rancher服务)：

```
apiVersion: traefik.containo.us/v1alpha1
kind: TraefikService
metadata:
  name: traefik-webui-traefikservice
  namespace: kube-system

spec:
  weighted:
    services:
      - name: traefik-ingress-service
        weight: 1
        port: 8080

---
apiVersion: traefik.containo.us/v1alpha1
kind: TraefikService
metadata:t
  name: rancher-traefikservice
  namespace: cattle-system

spec:
  weighted:
    services:
      - name: rancher
        weight: 1
        port: 80
```

#### IngressRouter配置

* 定义强制https的redirect-https,rancher的ingressrouter可以直接引用
* 配置X-Forwarded-Proto的头信息
* 配置IngressRouter,并设置TLS证书通过secretName引入, 需要提前在kubernetes配置secret，上一章已经记录这里先忽略了

```
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
  namespace: kube-system
spec:
  redirectScheme:
    scheme: https

---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: svc-rancher-headers
  namespace: cattle-system
spec:
  headers:
    customRequestHeaders:
      X-Forwarded-Proto: "https"

---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-webui
  namespace: kube-system
spec:
  entryPoints:
  - admin
  routes:
  - match: Host(`traefik-cicd.test.cn`)
    kind: Rule
    services:
    - name: api@internal
      kind: TraefikService
      namespaces: kube-system

---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: rancher
  namespace: cattle-system
spec:
  entryPoints:
  - http
  routes:
  - match: Host(`rancher-cicd.test.cn`)
    kind: Rule
    services:
    - name: rancher-traefikservice
      kind: TraefikService
      namespaces: cattle-system
      port: 80
    middlewares:
    - name: redirect-https

---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: rancher-tls
  namespace: cattle-system
spec:
  entryPoints:
  - https
  routes:
  - match: Host(`rancher-cicd.test.cn`)
    kind: Rule
    services:
    - name: rancher-traefikservice
      kind: TraefikService
      namespaces: cattle-system
      port: 443
    middlewares:
    - name: svc-rancher-headers
  tls:
    secretName: tls-rancher-ingress
    options:
      name: mytlsoption
      namespaces: cattle-system
    stores:
      default:
        defaultCertificate:
          certFile: "/ssl/tls.crt"
          keyFile: "/ssl/tls.key"
```

#### Traefik2.1部署(DeployMent)

* 配置traefik service服务 
* 创建traefik configmap,并配置entrypoints、provider
* Deployment方式部署traefik ingress controller


```
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
    - protocol: TCP
      port: 80
      nodePort: 23456
      name: http
    - protocol: TCP
      port: 443
      nodePort: 23457
      name: https
    - protocol: TCP
      port: 8080
      nodePort: 22180
      name: admin
  type: NodePort

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: traefik-conf
  namespace: kube-system
data:
  traefik.toml: |
    [global]
      checkNewVersion = false
      sendAnonymousUsage = false

    [log]
      level = "DEBUG"

    [api]
      dashboard = true

    [metrics.prometheus]
      buckets = [0.1,0.3,1.2,5.0]
      entryPoint = "metrics"

    [entryPoints]
      [entryPoints.http]
        address = ":80"
      [entryPoints.https]
        address = ":443"
      [entryPoints.admin]
        address = ":8080"

    [providers]
      [providers.kubernetesCRD]
      #[providers.kubernetesIngress]
      #[providers.rancher]
      #[providers.docker]

---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: traefik
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: traefik-ingress-lb
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      nodeSelector:
        node-role.kubernetes.io/traefik: "true"
      volumes:
      - name: ssl
        secret:
          secretName: traefik-cert
      - name: config
        configMap:
          name: traefik-conf
      containers:
      - image: traefik:v2.1
        name: traefik-ingress-lb
        volumeMounts:
        - mountPath: "/ssl"
          name: "ssl"
        - mountPath: "/config"
          name: "config"
        resources:
          limits:
            cpu: 1000m
            memory: 800Mi
          requests:
            cpu: 500m
            memory: 600Mi
        ports:
        - name: http
          containerPort: 80
          hostPort: 80
        - name: https
          containerPort: 443
          hostPort: 443
        - name: admin
          containerPort: 8080
        args:
        - --configfile=/config/traefik.toml
        - --entrypoints.http.Address=:80
        - --entrypoints.https.Address=:443
        - --entrypoints.https.Address=:8080
        - --api
        - --accesslog
```

#### 部署traefik2.1到Kubernetes集群

```
kubectl apply -f 01-crd.yaml
kubectl apply -f 02-rbac.yaml
kubectl apply -f 03-tlsoption.yaml
kubectl apply -f 04-traefikservices.yaml
kubectl apply -f 05-ingressrouter.yaml
kubectl apply -f 06-traefik.yaml
```

更多配置信息，请移步我的github仓库：https://github.com/iwz2099/kubecase