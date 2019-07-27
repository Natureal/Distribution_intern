## Kubernetes Docs

**Notice:**

	[]：表示可选的字段

	<>：表示必须填写的字段
**References**

k8s官方安装指南：
https://kubernetes.io/docs/setup/independent/install-kubeadm/

k8s官方基础tutorial：
https://kubernetes.io/docs/tutorials/kubernetes-basics/

kubectl命令中文文档：
http://docs.kubernetes.org.cn/683.html

Flannel网络原理讲解：
https://blog.laputa.io/kubernetes-flannel-networking-6a1cb1f8ec7c

Kubernets调用GPU：
https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/
https://github.com/NVIDIA/k8s-device-plugin

#### Outline – 启动K8S集群

（1）安装Docker

（2）按照官方安装指南，安装kubeadm, kubelet and kubectl

（3）关闭交换空间，参照第二条

（4）初始化K8S Master，参照第三条，pod network使用Flannel

（5）在Worker节点上安装K8S，并join到集群上，参照第八条

（6）kubectl get nodes查看各个节点的状态，都为ready表示集群搭建完成。

**1. Basic components**

① kubeadm: 引导集群的启动、重启等

② kubelet: (1) 集群中每台机器中都有 (2) 用于启动pods和container等

③ kubectl: 与集群对话，操作集群的工具

**2. Kubernetes 基本要求**

① 关闭交换空间：

```
→ sudo swapoff -a
```

Addition: 查看目前交换空间的命令：

→ free

（另一种关闭交换空间的方式：打开/etc/fstab，将含有swap的那一行注释掉）

**3. Kubernetes 初始化（包括重启之后）**

```
→ sudo kubeadm reset

→  sudo -E kubeadm init --pod-network-cidr=10.244.0.0/16

（pod network的设置基于使用Flannel的情况）

→ sudo cp /etc/kubernetes/admin.conf $HOME/

→ sudo chown $(id -u):$(id -g) $HOME/admin.conf

→ export KUBECONFIG=$HOME/admin.conf

(这句写到.bashrc里去）

→ sudo -E kubectl apply -f https://raw.githubusercontent

.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml
```

以下的配置基于使用spark的情况

创建spark service account：

→ kubectl create serviceaccount spark

→ kubectl create clusterrolebinding spark-role --clusterrole=edit --serviceaccount=default:spark --namespace=default

**4. Kubectl 命令用法**

① 查看cluster信息
```
→ kubectl cluster-info [dump] # dump is for debug
```

② 查看资源

```
→ kubectl get <resource_name>
examples:
→ kubectl get nodes [-o wide] # 查看各个节点[详细]信息
→ kubectl get pods [-o wide] # 查看各个pods[详细]信息
→ kubectl get services [-o wide] # 查看各个服务[详细]信息，services可简写为svc
→ kubectl get get developments [-o wide]
    # 查看各个部署[详细]信息，deployments可简写为deploy 或ds
```

③ 描述对象（查看其详细配置信息）
```
→ kubectl describe <resource_instance_id>
examples:
→ kubectl describe <pod__id> # 描述一个pod
→ kubectl describe <node_id> # 描述一个node
```

④ 查看对象的日志
```
→ kubectl logs [-f] [-p] <pod_id> [-c <constainer>]
[-f]: follow，持续输出
[-p]: previous，可输出已经终止的容器的日志
[-c]: container，指定pod内的某个容器

examples:
→ kubectl logs -f <pod_id> # 持续输出某pod的日志
```

⑤ 删除所有pods
```
→ kubectl delete --all pods [--namespace=<name of namespace>]
```

⑥ 运行一个镜像的pod，并生成其deployment
```
→ kubectl run [options] <DEPLOY_NAME>
   --image=[registry_ip:port/]<image_name:version>
   --port=<pod_port> --command <command>
```
example:
```
→ kubectl run  -it zsl --image=nvidia/cuda:py2.7-torch0.3-cuda9.0
   --port=80 bash
```

⑦ 将deployment暴露为服务
```
→ kubectl expose deployment/<deploy_name> --type=Nodeport
    [--name=<SERVICE_NAME>] --port=<service_port> [--target-port=<pod_port>]
```

⑧ 让deployment根据负载自动扩容缩容
```
→ kubectl autoscale deployment <deploy_name> [--min=<num>] --max=<num>
```

⑨ 连接到在run的pod（container）
```
→ kubectl attach <pod_id> -c <container_name>
```

**5. Kubernets 端口映射**
```
→ kubectl port-forward <pod_id> <port1>:<port2>
```

**6. 让Master节点也能被分配任务（pod）**
```
→ kubectl taint nodes --all node-role.kubernetes.io/master-
```

**7. 封锁某个节点，使其不能被分配任务（pod）**
```
→ kubectl cordon <node_name>
解除封锁：
→ kubectl uncordon <node_name>
```

**8. 添加新的Node（join失败就重启一下节点）**

① 在新节点上安装kubeadm

执行kubeadm reset

关闭交换空间

② join的命令用法：
```
→ kubeadm join --token <token> <master_ip:port>
    --discovery-token-ca-cert-hash sha256:<hash>
```
example:
```
→ kubeadm join 10.232.140.53:6443 --token e6efmg.mfncukt4l2eglzcf
    --discovery-token-ca-cert-hash
    sha256:21e46e14f78672479b8a64aa63c9b6311f7e3eac5c7be148d3be1e086ddcb245
```

③ 查看master ip 和port：
```
→ kubectl cluster-info
```

④ 创建和查看token：
```
→ kubeadm token create
（join之前需要创建一个新的token，有效期24h）
→ kubeadm token list
```

⑤ 查看cluster的sha256哈希值：
```
→ openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt |
openssl rsa -pubin -outform der 2>/dev/null |
openssl dgst -sha256 -hex | sed 's/^.* //'
```

**9. 为K8S配置GPU Support**

参考https://github.com/NVIDIA/k8s-device-plugin

①更改/etc/docker/daemon.json

②
```
kubectl create -f
 https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v1.11/nvidia-device-plugin.yml
```

③ 查看新部署的 nvidia plugin daemonset
```
→ kb get ds --all-namespaces
```
