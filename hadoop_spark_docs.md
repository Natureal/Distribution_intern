## Hadoop & Spark Docs

**Notice:**

[]：表示可选的字段

<>：表示必须填写的字段

#### 一． Hadoop集群搭建

1. 修改每个节点IP和名字

```
/etc/hostname
/etc/hosts
```

2. SSH 无密登录

（1） 在集群每个节点上修改 /etc/ssh/sshd_config

将下面几行前面的”#”注释取消掉：

```
RSAAuthentication yes

PubkeyAuthentication yes

AuthorizedKeysFile  %h/.ssh/authorized_keys
```

（2） 在每个节点上运行：

```
--> ssh-keygen –t rsa –P ''

--> cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

--> cd ~/.ssh

--> chmod 600 authorized_keys

--> ssh-add   ~/.ssh/id_rsa

--> service ssh restart
```

（3） 使用ssh-copy-id命令将公钥传送到远程主机上

在tong-dl（master）上

```
--> ssh-copy-id peng@deepwork-1

--> ssh-copy-id peng@deepwork-2
```

在其他节点上

```
--> ssh-copy-id peng@tong-dl
```

（4） 在每个节点上配置JAVA环境变量

如tong-dl：

```
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64

export PATH="$JAVA_HOME/bin:$PATH"

export JRE_HOME=${JAVA_HOME}/jre

export CLASS_PATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
```

使配置生效

（5） 下载解压hadoop-2.7.7路径：/usr/local/

配置环境变量HADOOP_HOME：

```
export HADOOP_HOME=/usr/local/hadoop-2.7.7

export PATH=$PATH:$HADOOP_HOME/bin
```

（6） 配置hadoop配置文件（详见tong-dl上的配置）

进入/usr/local/hadoop-2.7.7/etc/hadoop

I. 在hadoop-env.sh中加入JAVA_HOME

II. 在core-site.xml中配置master的地址端口，以及临时文件存储路径。

```
#Use example:path = 'hdfs://tong-dl:9000/user/peng/fileBank'
```

III. 在hdfs-site.xml中配置namenode地址端口，以及数据副本数量。

IV. 修改好了之后将hadoop文件夹复制到每个节点上。

V. 记得修改每个节点上关于JAVA_HOME的配置，因为每个节点的java路径可能不一致。（如deepwork-1和tong-dl就不一样）

（7） 格式化hdfs，清空数据（在master上）

```
--> hadoop namenode -format
```

清空每个节点的数据文件夹：

```
${HADOOP_HOME}/tmp/dfs
```

（8） 启动hadoop集群（缩写命令）

--> hp-start

（9） 关闭hadoop集群

--> hp-stop

二． Spark集群搭建

（1） 在每个节点上下载解压spark，在		/usr/local/spark-2.3.2

（2） 将~/.bashrc中的环境变量，SPARK_HOME版本改成2.3.2

```
export SPARK_HOME=/usr/local/spark-2.3.2

export PATH=$PATH:$SPARK_HOME/bin
```

（3） 在每个节点上修改spark配置

进入：/usr/local/spark-2.3.2/conf

I. 在spark-env.sh中添加：

（如在tong-dl上：）
```
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export HADOOP_HOME=/usr/local/hadoop-2.7.7
export HADOOP_CONF_DIR=/usr/local/hadoop-2.7.7/etc/hadoop
export LD_LIBRARY_PATH=$HADOOP_HOME/lib/native
```

（每个节点上的JAVA路径可能不一样）

II. 在slaves中添加：

（在tong-dl上：）
deepwork-1
deepwork-2

（4） 启动spark集群

--> spark-start

（5） 关闭spark集群

--> spark-stop


Commands:

peng@tong-dl:~/CHEN_PENG_Projects/XGBoost/distributed_workshop/data$ hdfs dfs -put vec_train_data.csv /xgboost
