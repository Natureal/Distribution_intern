#### Document records

- written during the internship in Hitachi Central Laboratory

- Doc List

  - [Kubernetes Docs](https://github.com/Natureal/Distribution_intern/blob/master/kubernetes_docs.md)
  - [Hadoop & Spark](https://github.com/Natureal/Distribution_intern/blob/master/hadoop_spark_docs.md)
  - [File Tree](https://github.com/Natureal/Distribution_intern/blob/master/file_tree_docs.md)
  - [Project1 Spark on K8S](https://github.com/Natureal/Distribution_intern/blob/master/Proj1_spark_on_k8s_docs.md)
  - [Project1 Steps](https://github.com/Natureal/Distribution_intern/blob/master/Proj1_steps_docs.md)

---

#### Problems in using Docker

**I. Building Docker Private Repository** ======================================

Motivation: 镜像除了从 dockerhub 中 pull，还可以自己搭建私有库，以提供自定义的镜像供分布式系统中不同 Node 使用。

1. Pull image "registry" from dockerhub

```
sudo docker pull registry
```

2. Run the private repository persistently on the port 6001

```
sudo docker run -d -p 6001:5000 --name [name] --restart=always --privileged=true registry
```

3. Need some configuration to let K8S pull images from private repo by default.

缺点: 缺少 UI，难以访问，需要二次开发。

**II. Nvidia Docker** ======================================================

配置支持 nvidia 驱动的 docker 需要专门安装 nvidia-docker。

**III. Proxy for Docker** ===================================================

为 Docker 配置 Proxy，用于 pull 官网镜像等。

1. Configuration

在/etc/systemd/system/docker.service.d/目录下添加 http-proxy.conf 以及 https-proxy.conf

文件内容 examples：

http-proxy.conf:
```
[Service]

Environment="HTTP_PROXY=http://<username>:<password>@xxxx:8080"

Environment="http_proxy=http://<username>:<password>@xxxx:8080"
```

https-proxy.conf:
```
[Service]

Environment="HTTPS_PROXY=http://<username>:<password>@xxxx:8080"

Environment="https_proxy=http://<username>:<password>@xxxx:8080"
```

2. After changes
```
Flush: sudo systemctl daemon-reload

Restart: sudo systemctl restart docker
```

3. Check the environment parameters
```
sudo systemctl show --property=Environment docker
```

**IV. Proxy and DNS for Containers** =========================================

为 Docker 容器配置 Proxy 和 DNS，用于在 container 中 apt-get 等联网操作。

1. Proxy for Containers

方法1：在Dockerfile中添加ENV
```
ENV http_proxy http://<username>:<password>@xxxx:8080

ENV https_proxy http://<username>:<password>@xxxx:8080
```

方法2：进入 container 里面更改环境变量（写在.bashrc里）

2. DNS for Containers

方法1：

进入 container 进行配置，在 /etc/resolv.conf 里添加：
```
nameserver 133.144.xxx.xxx

nameserver 158.214.xxx.xxx
```

方法2：

写在 ~/.bashrc 里：
```
echo “nameserver 133.144.xxx.xxx” > /etc/resolv.conf

echo “nameserver 158.214.xxx.xxx” > /etc/resolv.conf
```

**V. 封装 Spark 组件与 Pytorch 算法模型** =======================================

**版本: Spark 2.3.2**

这是项目中遇到的最大难点，解决它也能更深入了解 Dockerfile 的机制！

还得从合并 Spark 组件和 Pytorch 算法模型说起...

1. Spark 的 Dockerfile 中第一句是

```
FROM openjdk:8-alpine
```

2. 算法模型是基于已有 image 建立的（其根镜像是 ubuntu）

```
FROM nvidia/cuda：py2.7-torch0.3-cuda9.0
```

3. 那么问题来了，要组合这两个容器，一个 Dockerfile 要出现两个 FROM，合理吗？

紧接着，我发现了多阶段构建这个东西，允许出现多个 FROM。其原理其实很简单，将一个阶段的文件复制到另外一个阶段，在最终的镜像中保留下你需要的内容。

Spark Dockerfile 中的 openjdk:8-alpine 镜像是基于 alpine 镜像的，alpine 是一个轻量级的 Linux 发行版。如果多阶段构建的话，假设先继承一个 FROM openjdk:8-alpine，并构建 Spark，接着需要拷贝哪些文件到下一阶段呢，感觉有点复杂。

进一步研究发现 Spark 的 Dockerfile 实际上也就是在 Linux 的基础上导入了一些依赖库（比如 JRE 等），算法模型的 image 也是基于 Linux(ubuntu) 的，那么考虑在其基础上构建 Spark，创建一个新的 Dockerfile 继承算法模型 FROM nvidia/cuda：py2.7-torch0.3-cuda9.0，接着在 Dockerfile 中 Copy Spark 所需要的组件文件夹，执行相关操作（如导入依赖）。

观察 Spark 2.3.2 的 Dockerfile 的每一部分：
```shell
FROM openjdk:8-alpine

ARG spark_jars=jars
ARG img_path=kubernetes/dockerfiles
```

（1）FROM 后面跟着基础镜像。

（2）ARG 为命名临时变量。

```shell
RUN set -ex && \
    apk upgrade --no-cache && \
    apk add --no-cache bash tini libc6-compat && \
    mkdir -p /opt/spark && \
    mkdir -p /opt/spark/work-dir \
    touch /opt/spark/RELEASE && \
    rm /bin/sh && \
    ln -sv /bin/bash /bin/sh && \
    chgrp root /etc/passwd && chmod ug+rw /etc/passwd
```
（1）RUN 为执行 shell 命令。

（2）set -ex，用于设置 shell；-e: 若指令传回值不等于0，则立即退出shell；-x: 执行指令后，会先显示该指令及所下的参数

（3）apk 是 alpine 提供的软件包管理工具

（4）upgrade --no-cache 重新更新已安装的软件包

（5）add 命令从仓库中安装最新软件包，并自动安装必须的依赖包

（6）mkdir 创建文件夹

（7）ln -sv /bin/bash /bin/sh 放弃 Dash，只用 Bash

（8）chgrp 命令用于变更文件或目录的所属群组

（9）chmod ug+rw 与用户属于同一组的用户加读写权限

```shell
COPY ${spark_jars} /opt/spark/jars
COPY bin /opt/spark/bin
COPY sbin /opt/spark/sbin
COPY conf /opt/spark/conf
COPY ${img_path}/spark/entrypoint.sh /opt/
COPY examples /opt/spark/examples
COPY data /opt/spark/data

ENV SPARK_HOME /opt/spark

WORKDIR /opt/spark/work-dir

ENTRYPOINT [ "/opt/entrypoint.sh" ]
```
（1）COPY

（2）ENV

（3）WORKDIR

（4）ENTRYPOINT



---

#### Problems in using Kubernetes

****


#### Features of HDFS

1. 概念

HDFS 是基于操作系统本身的文件系统之上的虚拟文件系统。

- 最小的单元时 block，默认 64MB。

- 一个文件被分成若干块，可以存储在不同的机器上（Datanode）。

- HDFS 结构：Master / Slave 架构

  - 一个 NameNode

  - 若干 DataNode

  - （一个 Secondary NameNode）
