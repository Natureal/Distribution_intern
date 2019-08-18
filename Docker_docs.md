##Docker Docs

**Notice:**

	[]：表示可选的字段

	<>：表示必须填写的字段

Docker官方Tutorial：
https://docs.docker.com/install/linux/docker-ce/ubuntu/#os-requirements

Docker中文文档：
https://docs.docker-cn.com/get-started/

Docker官方image hub
https://hub.docker.com/explore/

---

#### 1. 启动Docker

```
→ sudo systemctl start docker

→ sudo service docker start
```

**查看docker的状态：**

```
→ sudo systemctl status docker
```

#### 2. 创建image
```
→ sudo docker build -t <name> <path>
```

example:
```
1. sudo docker build -t hello .
2. sudo docker build -t good ~/Dockerfile
```

#### 3. 查看本地images
```
→ sudo docker images
```

#### 4. 查看本地在run的containers
```
→ sudo docker ps [-a]
```

#### 5. 基于在run的container，生成一个新image
```
→ sudo docker commit <container_id> <name[:tag]>
example:
→ sudo docker commit c3f279d17e0a testimage:version3
```

#### 6. Docker的配置产生更改，需要刷新docker服务
```
→ sudo systemctl daemon-reload
→ sudo systemctl restart docker
```

#### 7. 查看目前docker的环境变量
```
→ sudo systemctl show --property=Environment  docker
```

#### 8. Run一个image，生成container
```
→ sudo docker run [options] <image_name> <command>
options:
(1) -d 后台运行
(2) -p 指定端口，e.g. -p x:y, 主机的x映射到y
(3) -t 分配终端
(4) -i 以交互模式运行
(5) --dns  指定dns
(6) –name指定容器名字
```

example:
```
→ sudo docker run -p 5000:80 -it ubuntu_sample /bin/bash
```

在容器内部开一个新终端：
```
→ sudo docker exec -it <container_id> /bin/bash
```

#### 9. 停止容器
```
→ sudo docker stop <container_id>
```

#### 10. 删除镜像
```
→ sudo docker rmi <image_name:tag>
```

#### 11. 删除所有的<none>镜像
```
sudo docker rmi $(sudo docker images -f "dangling=true" -q)
```

#### 12. 为镜像添加tag
```
→ sudo docker tag <source_image> <tagged_image>
example:
→ docker tag busybox 127.0.0.1:5000/busybox
```

#### 13. Docker的Proxy以及DNS配置（用于pull官网镜像等）

**part1：proxy配置**

在/etc/systemd/system/docker.service.d/目录下添加

http-proxy.conf 以及https-proxy.conf

文件内容：

http-proxy.conf:
```
[Service]
Environment="HTTP_PROXY=http://<username>:<password>@obprx.intra.hitachi.co.jp:8080"
Environment="http_proxy=http://<username>:<password>@obprx.intra.hitachi.co.jp:8080"
```

https-proxy.conf:
```
[Service]
Environment="HTTPS_PROXY=http://<username>:<password>@obprx.intra.hitachi.co.jp:8080"
Environment="https_proxy=http://<username>:<password>@obprx.intra.hitachi.co.jp:8080"
```

--------------------------------------------------------------------------------

※另一种配置proxy的方法，原理都是添加docker的环境变量（在新版本中不推荐，但可用）：

在/etc/default/docker中添加如下两行：
```
export http_proxy=http://<username>:<password>@obprx.intra.hitachi.co.jp:8080
export https_proxy=http://<username>:<password>@obprx.intra.hitachi.co.jp:8080
```

**part2：DNS配置**

方法1：

在/etc/default/docker中添加如下一行：
```
DOCKER_OPTS="--dns 133.144.247.102 --dns 158.214.50.44 --dns 8.8.8.8 --dns 8.8.4.4"
```
方法2：

在/lib/systemd/system/docker.service中的ExecStart那一行添加红色字段：
```
ExecStart=/usr/bin/dockerd --dns=133.144.247.102 --dns=158.214.50.44
```

#### 14. 为Docker的container配置Proxy以及DNS（用于在container中apt-get等联网操作）

**part1：Proxy配置**

方法1：在Dockerfile中添加ENV
```
ENV http_proxy http://71477685:pznaiie05e@obprx.intra.hitachi.co.jp:8080
ENV https_proxy http://71477685:pznaiie05e@obprx.intra.hitachi.co.jp:8080
```

方法2：进入container里面更改环境变量（写在.bashrc里）

**part2：DNS配置**

进入container进行配置：

在/etc/resolv.conf里添加：
```
nameserver  133.144.247.102
nameserver 158.214.50.44
```

或者写在~/.bashrc里：
```
echo “nameserver 133.144.247.102” > /etc/resolv.conf
echo “nameserver 158.214.50.44” > /etc/resolv.conf
```

#### 15. Docker 本地私有库

Docker默认从官网的镜像仓库里pull镜像，我们也可以建立属于自己的仓库。

（1）首先从官网pull一个用于建立仓库的镜像
```
sudo docker pull registry
```

（2）run一个自动重启的仓库
```
sudo docker run -d -p 6001:5000 [--name <NAME>] --restart=always –privileged=true registry
```

（3）到此为止，用sudo docker ps -a能看到在run的仓库，并且是run在<IP>的6001端口上。

查看仓库内镜像：<IP>:6001/v2/_catalog

查看具体镜像信息：<IP>6001/v2/<image_name>/tags/list

（4）删除仓库内镜像的方法：

首先进入仓库：sudo docker exec -it <仓库ID> sh

再进入/var/lib/registry/docker/registry/v2/repositories/，就能看到各个镜像的文件夹，用rm -r <image_to_be_deleted>删除。

（5）当我们想让Docker默认从私有仓库pull镜像时（比如在用Kubernets时，Kubernets基于docker去pull镜像，当我们希望Kubernets中不同Node默认使用私有库的镜像时）：

需要在/lib/systemd/system/docker.service中的ExecStart那一行添加蓝色字段：
```
ExecStart=/usr/bin/dockerd -H fd:// --insecure-registry 10.232.140.53:6001 --dns=133.144.247.102 --dns=158.214.50.44 --dns=8.8.8.8 --dns=8.8.4.4
```

然后在/etc/docker/daemon.json的末尾加入（不知是不是必须，加了再说）：
```
{
	"insecure-registries": [
		"10.232.140.53:6001"
	]
}
```


然后重启docker服务，方法见6

#### 16. Nvidia-docker 配置

（1）安装nvidia-docker

（2）更改/etc/docker/daemon.json的default-runtime代码块为：
```
{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
```

（3）重启docker，方法见6

#### 17. Docker 配置文件位置较多，以下是汇总

1. /etc/default/docker
2. /lib/systemd/system/docker.service
3. /etc/systemd/system/docker.service.d/*
4. /etc/docker/daemon.json

修改配置文件后重启docker，方法见6
