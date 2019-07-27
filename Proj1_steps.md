## Project1.Spark on K8S Steps

**Notice:**

1. []：表示可选的字段

2. <>：表示必须填写的字段

**1. 基础准备**

所有简写的命令都可在 ~/.myalias.sh 中查阅。

**2. 检查Docker的proxy**

参考Docker文档第13条

**3. 检查Docker 私有库在run**
```
--> sdcxx
```
发现有一个run在6001端口上的名为registry的容器，OK
（如果没有，需要配置私有库，参考Docker文档第15条）

**4. 检查Docker私有库中的镜像**
```
--> sdi | grep 10.232.140.53:6001
```

检查最新版的镜像的存在：

```
10.232.140.53:6001/spark-driver:1027-cmd

10.232.140.53:6001/spark-executor:1027-cmd
```

**5. 启动K8S集群**

（1） 首先检查k8s是否已启动（机器重启后需要重启k8s）
```
--> kbn
```

（2） 启动k8s

参考K8S文档第3条

（3） 刚启动就一个节点（tong-dl），可以join其他节点

参考K8S文档第8条

（4） 将master节点设为“可被调度状态”，否则master不跑pods

参考K8S文档第6条

（5） 用kbn命令查看集群节点，各个节点状态都应该为“Ready”

**6. 启动Spark on K8S**

进入~/CHEN_PENG_Projects
```
--> source run_clusterZSL.sh
--> kbp （查看被创建的driver和executors）
```

**7. 启动Kafka**
```
--> zookeeper-start
--> kafka-start
```

**8. 启动Web GUI**

（1）启动上传图片的网页

进入~/CHEN_PENG_Projects/clusterWebGUI
```
--> py app.py
```

（2）启动产生结果的网页（2s刷新一次）

进入~/CHEN_PENG_Projects/clusterWebResult
```
--> py result.py
```

**9. 使用demo**

访问10.232.140.53:31745上传图片

访问10.232.140.53:31746查看结果
