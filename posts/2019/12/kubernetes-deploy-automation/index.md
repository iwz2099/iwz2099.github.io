# 自动化部署k8s集群


#### 环境准备

* 架构:

![架构图](https://github.com/easzlab/kubeasz/blob/master/pics/ha-2x.gif)

* Master节点
```
172.16.244.14
172.16.244.16
172.16.244.18
```
* Node节点
```
172.16.244.25
172.16.244.27
```

* Master节点VIP地址: 172.16.243.13

* 部署工具:Ansible/kubeasz


---

#### 初始化环境

1. 安装Ansible
```
apt update
apt-get install ansible expect
git clone https://github.com/easzlab/kubeasz
cp * /etc/ansible/
cd /etc/ansible
```
2. 配置ansible免密登录
ssh-keygen -t rsa -b 2048 #生成密钥
./tools/yc-ssh-key-copy.sh  hosts root 'rootpassword'

3. 准备二进制文件
```
cd tools
./easzup -D #默认情况都会下载到/etc/ansible/bin/目录下
```

4、配置hosts文件如下:
```
[kube-master]
172.16.244.14
172.16.244.16
172.16.244.18

[etcd]
172.16.244.14 NODE_NAME=etcd1
172.16.244.16 NODE_NAME=etcd2
172.16.244.18 NODE_NAME=etcd3

#haproxy-keepalived
[haproxy]
172.16.244.14
172.16.244.16
172.16.244.18

[kube-node]
172.16.244.25
172.16.244.27


# [optional] loadbalance for accessing k8s from outside
[ex-lb]
172.16.244.14 LB_ROLE=backup EX_APISERVER_VIP=172.16.243.13 EX_APISERVER_PORT=8443
172.16.244.16 LB_ROLE=backup EX_APISERVER_VIP=172.16.243.13 EX_APISERVER_PORT=8443
172.16.244.18 LB_ROLE=master EX_APISERVER_VIP=172.16.243.13 EX_APISERVER_PORT=8443

# [optional] ntp server for the cluster
[chrony]
172.16.244.18

[all:vars]
# --------- Main Variables ---------------
# Cluster container-runtime supported: docker, containerd
CONTAINER_RUNTIME="docker"

# Network plugins supported: calico, flannel, kube-router, cilium, kube-ovn
#CLUSTER_NETWORK="flannel"
CLUSTER_NETWORK="calico"

# Service proxy mode of kube-proxy: 'iptables' or 'ipvs'
PROXY_MODE="ipvs"

# K8S Service CIDR, not overlap with node(host) networking
SERVICE_CIDR="10.68.0.0/16"

# Cluster CIDR (Pod CIDR), not overlap with node(host) networking
CLUSTER_CIDR="10.101.0.0/16"

# NodePort Range
NODE_PORT_RANGE="20000-40000"

# Cluster DNS Domain
CLUSTER_DNS_DOMAIN="cluster.local."

# -------- Additional Variables (don't change the default value right now) ---
# Binaries Directory
bin_dir="/opt/kube/bin"

# CA and other components cert/key Directory
ca_dir="/etc/kubernetes/ssl"

# Deploy Directory (kubeasz workspace)
base_dir="/etc/ansible"
```

---

#### 部署K8S集群

1. 初始化配置

```
cd /etc/ansible
ansible-playbook 01.prepare.yml
```
这个过程主要做三件事：
-  chrony role: 集群节点时间同步[可选]
-  deploy role: 创建CA证书、kubeconfig、kube-proxy.kubeconfig
-  prepare role: 分发CA证书、kubectl客户端安装、环境配置

2. 安装etcd集群
```
ansible-playbook 02.etcd.yml
```

3. 安装docker
```
ansible-playbook 03.docker.yml
```

4. 部署kubernetes master
```
ansible-playbook 04.kube-master.yml
```

5. 部署kubernetes node
```
ansible-playbook 05.kube-node.yml
```

6. 部署kubernetes network(这里选择calico)
```
ansible-playbook 06.network.yml
```

7. 部署ingress/k8s dashbaord/coredns

> 配置ingress所用ssl证书, 这里仓库默认使用的traefik1.7.12,后边我们打算升级为2.0
```
kubectl create secret tls traefik-cert --key=test.cn.key --cert=test.cn.pem  -n kube-system
secret/traefik-cert created  
```
> 部署集群扩展
```
ansible-playbook 07.cluster-addon.yml
```

8. master节点取消污点
> 默认部署完后，master节点状态是打了污点，不在调度策略如下：
```
# kubectl  get node
NAME            STATUS                     ROLES    AGE   VERSION
172.16.244.14   Ready,SchedulingDisabled   master   91m   v1.16.2
172.16.244.16   Ready,SchedulingDisabled   master   91m   v1.16.2
172.16.244.18   Ready,SchedulingDisabled   master   91m   v1.16.2
172.16.244.25   Ready                      node     90m   v1.16.2
172.16.244.27   Ready                      node     90m   v1.16.2
# kubectl  describe node  172.16.244.14 |grep Taint
Taints:             node.kubernetes.io/unschedulable:NoSchedule
```

由于机器资源有限，所以把master也加入调度可用状态
```
# kubectl patch node 172.16.244.14 -p '{"spec":{"unschedulable":false}}'
# kubectl  get node
NAME            STATUS   ROLES    AGE   VERSION
172.16.244.14   Ready    master   95m   v1.16.2
172.16.244.16   Ready    master   95m   v1.16.2
172.16.244.18   Ready    master   95m   v1.16.2
172.16.244.25   Ready    node     94m   v1.16.2
172.16.244.27   Ready    node     94m   v1.16.2
```

---

#### 部署外部负载均衡器(Keepalived+Haproxy)
```
ansible-playbook   roles/ex-lb/ex-lb.yml
```

---

#### 安装Helm3
> 参考https://rancher.com/docs/rancher/v2.x/en/installation/ha/helm-rancher/

```
cd /opt/soft
wget https://get.helm.sh/helm-v3.0.1-linux-amd64.tar.gz
tar xf helm-v3.0.1-linux-amd64.tar.gz
cd linux-amd64/
cp helm  /usr/local/bin/
```

---

#### 部署Rancher

1. 创建证书
> 因为由自己域名证书,所以这里使用k8s的secret创建的证书,当然也可以使用cert-manager工具签发rancher自己的证书或者使用letsEncrypt
```
kubectl -n cattle-system create secret tls tls-rancher-ingress --cert=test.cn.pem  --key=test.cn.key
```
2. 安装rancher
```
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher-cicd.test.cn --set ingress.tls.source=secret
```

3. 检查ingress、资源和部署状态
```
# kubectl  get ingress  --all-namespaces
NAMESPACE       NAME      HOSTS                   ADDRESS   PORTS     AGE
cattle-system   rancher   rancher-cicd.test.cn             80, 443   20h
# kubectl -n cattle-system rollout status deploy/rancher
Waiting for deployment "rancher" rollout to finish: 0 of 3 updated replicas are available...
Waiting for deployment "rancher" rollout to finish: 1 of 3 updated replicas are available...
Waiting for deployment "rancher" rollout to finish: 2 of 3 updated replicas are available...
deployment "rancher" successfully rolled out
# kubectl -n cattle-system get deploy rancher
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
rancher   3/3     3            3           5m5s
```

至此，整个K8S集群已经搭建完毕，如果顺利的话，整个过程应该在10分钟左右，重要的是提前规划好集群。
