# kubernetes集群添加用户


之前通过ansible搭建了kubernetes集群环境,这里需求主要是添加一个用户进行日常管理，并限制到指定的namespace，接下来进行操作：

#### 生成证书

准备证书生成文件csr,并通过kubernetes的ca签发证书

```
# vim cicd-admin-csr.json
{
  "CN": "cicd-admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes  cicd-admin-csr.json | cfssljson -bare cicd-admin
# ls -l cicd-admin*
-rw-r--r-- 1 root root 1001 Dec 19 16:51 cicd-admin.csr
-rw-r--r-- 1 root root  224 Dec 19 16:50 cicd-admin-csr.json
-rw------- 1 root root 1675 Dec 19 16:51 cicd-admin-key.pem
-rw-r--r-- 1 root root 1387 Dec 19 16:51 cicd-admin.pem
```

#### 设置集群参数

```
export KUBE_APISERVER="https://cicd-k8s.test.cn:8443"
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/ssl/ca.pem \
--embed-certs=true \
--server=${KUBE_APISERVER} \
--kubeconfig=cicd-admin.kubeconfig

```

#### 设置客户端认证参数

```
kubectl config set-credentials cicd-admin \
--client-certificate=/etc/kubernetes/ssl/cicd-admin.pem \
--client-key=/etc/kubernetes/ssl/cicd-admin-key.pem \
--embed-certs=true \
--kubeconfig=cicd-admin.kubeconfig
```


#### 设置上下文参数
```
kubectl config set-context kubernetes \
--cluster=kubernetes \
--user=cicd-admin \
--namespace=cicd \
--kubeconfig=cicd-admin.kubeconfig
```

#### 设置默认上下文

```
kubectl config use-context kubernetes --kubeconfig=cicd-admin.kubeconfig
```

#### 客户端使用
将kubeconfig文件覆盖~/.kube/config
```
cp cicd-admin.kubeconfig ~/.kube/config
```

#### 绑定用户到指定namespace

```
kubectl create rolebinding cicd-admin-binding --clusterrole=admin --user=cicd-admin --namespace=cicd
kubectl create rolebinding cicd-admin-binding --clusterrole=admin --user=cicd-admin --namespace=chaos
```
