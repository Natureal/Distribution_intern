## Project1.Spark on K8S Docs

**Notice:**

1. []：表示可选的字段

2. <>：表示必须填写的字段

3. STEPS中涉及的镜像均已存在

4. 文中的“本机”均指10.232.140.53

5. 相关快捷命令记录在~/.myalias.sh中，如:

    alias sdi=‘sudo docker images’


**1. 基础准备**

（1）已有的基础镜像为nvidia/cuda：py2.7-torch0.3-cuda9.0

（2）完成搭建Docker本地私有仓库，查看当前在run的container能看到名为registry的容器。

（3）下载解压好spark-2.2.0，默认路径：/usr/local/spark-2.2.0

（4）在K8S上配置好spark的service account

（5）为K8S配置好GPU支持

（6）下载解压好Kafka，路径：/usr/local/kafka

（7）基于python2.7

**2.根据基础镜像构建新镜像**

（1）run一个nvidia/cuda：py2.7-torch0.3-cuda9.0镜像的container

（2）在container的hzrlfwl项目中添加新入口文件（在原来的入口文件上加上spark代码，用于调用spark），/home/hzrlfwl/webdemo/KafkaSparkZSL.py

（3）配置container中的spark环境（参考本机中的/usr/local/spark-2.2.0/dockerfiles/spark-base/Dockerfile）

I.在container中创建文件夹：/opt/spark，/opt/spark/work-dir)

II.在container中运行：
```
rm /bin/sh
ln -sv /bin/bash /bin/sh（为了将sh命令从默认的dash改为bash）
```

III.进入本机的/usr/local/spark-2.2.0路径，将jars，bin，sbin，conf文件夹拷贝到container中的/opt/spark/路径下

IV.配置container中的~/.bsahrc文件（如添加.proxy.sh等等）

V.将当前的container提交为新的镜像。（已经做好的镜像：nvidia/cuda:1026）

（4）基于上一步生成的镜像（nvidia/cuda:1026），进一步为其添加配置（这一步参考并结合了/usr/local/spark-2.2.0/dockerfiles/中关于driver和executor的Dockerfiles）：

进入本机/home/peng/Docker/my-final-spark-driver/路径，Dockerfile文件的作用为：根据nvidia/cuda:1026构建最终可用的spark driver镜像：nvidia/cuda:1026-cmd
```
→ sudo docker build -t nvidia/cuda:1026-cmd .
```

**Dockerfile内容简介：**

ENV添加image的环境变量

WORKDIR指定工作路径

CMD为启动容器时的执行命令

同理，进入本机/home/peng/Docker/my-final-spark-executor/路径，构建最终可用的spark executor镜像：
```
sudo docker build -t nvidia/cuda:1026-executor-cmd .
```

（5）将得到的两个新镜像push到本地私有库中：

I.更换tag：
```
sudo docker tag nvidia/cuda:1026-cmd 10.232.140.53:6001/spark-driver:1026-cmd
sudo docker tag nvidia/cuda:1026-executor-cmd 10.232.140.53:6001/spark-executor:1026-cmd
```

II.push到端口为6001的私有库中：
```
sudo docker push 10.232.140.53:6001/spark-driver:1026-cmd
sudo docker push 10.232.140.53:6001/spark-executor:1026-cmd
```

**3.运行Project**

（1）设定$SPARK_HOME

在运行之前，需要将系统的环境变量SPARK_HOME改成spark-2.2.0的路径，如：在~/.bashrc中：
```
export SPARK_HOME=/usr/local/spark-2.2.0
export PATH=$PATH:$SPARK_HOME/bin
```

原因：spark-2.3.2暂不支持Kubernets上的py应用，会遇到如下问题：Error: Python applications are currently not supported for Kubernetes.

（BTW，然而在用spark训练xgboost时，必须使用spark-2.3.0+，否则不支持xgboost4j）

（2）启动Zookeeper和Kafka集群

I.由于Kafka基于Zookeeper，先启动zookeeper（位于端口2191）： → zookeeper-start
	这里的zookeeper运行在本机2191端口上。

II.重开另一个终端启动Kafka集群：
```
→ kafka-start
```

III.查看Kafka中topics：
```
→ kftopics
```

IV.查看某个topic的详情：
```
→ kfdes <TOPIC_NAME>
```

V.在关闭时，运行

```
kafka-stop,zookeeper-stop
```

（3）运行镜像

进入本机/home/peng/IntegratedProjects/路径，运行脚本：

```
→ . run_clusterAutoZSL.sh
```

脚本参数：

I. deploy-mode: 部署模式，默认是本地，这里用 cluster

II. master: 指定Kubernets的master地址

III. kubernetes-namespace: 指定在Kubernets中工作的名空间

IV. jars: 指定附带的jar包

V. --conf: 指定spark参数

**Notice：**

spark.executor.instance指定executor数量，但是实际上会在run的executor取决于 Kafka topic 的分区数，尽量设置分区数等于executor数，使其一一对应。

VI. 最后一行指定运行的py脚本以及其参数

VII. py脚本的参数：

	--model：指定模型

	--cuda：用gpu训练

（4）查看spark on K8S运行状态

I.查看当前在运行的pods：
```
→ kbp
```

II.持续查看运行日志：

复制kbp运行后driver的NAME，运行：
```
→ kb logs -f <NAME_OF_DRIVER>
```

为了过滤掉太多spark的INFO信息，推荐：
```
→ kb logs -f <NAME_OF_DRIVER> | grep -v INFO
```

example：
```
→ kb logs -f zsl-1542091409316-driver | grep -v INFO）
```


（5）运行网页前端

I. 运行提交界面（先于项目运行）

进入本机/home/peng/IntegratedProjects/clusterWeb/
```
→ py app.py
```
（提交界面会将接受到的图片编码成base64，send到Kafka上名	为“zslStream”的topic中）

II. 运行结果界面

进入本机/home/peng/IntegratedProjects/clusterResultWeb/
```
→ py result.py
```
（结果界面从Kafka上名为“zslResult”的topic中接受结果(结	果由Spark集群上传)，并将其中的字段分段为：label，图片的	base64编码）

（6）提交图片，查看结果

I. 提交界面：10.232.140.53:31745

II. 结果界面：10.232.140.53:31746

  （结果界面没反应就刷新下）

（7) Optional：查看Spark集群状态：
```
→ kb port-forward <driver_pod_id> 4050:4040
```

这样就能在localhost:4050端口查看

（8）结束project

直接删除Kubernets上关于project的pods，如果只运行了project，可以删除所有pods：

```
→ kb delete --all pods
或者一个个删除
→ kb delete pod <POD_ID>
```

**4. 注意事项**

如果需要进一步修改项目里的代码，不能直接修改nvidia/cuda:1026-cmd或nvidia/cuda:1026-executor-cmd镜像，而应该修改nvidia/cuda:1026镜像，并重新生成两个～-cmd镜像。

原因：在建立～-cmd镜像时，需要通过Dockerfile导入
CMD命令，这些命令对于运行spark是必须的。如果直接在nvidia/cuda:1026-cmd的基础上commit一个新镜像，比如nvidia/cuda:1026-cmd2，会丢失原有的CMD命令，导致spark无法运行。

根本原因是Docker的commit机制不支持Dockerfile中CMD命令的迁移。
