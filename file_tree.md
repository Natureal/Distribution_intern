## File structure of projects

**Project1 Spark on K8S:**

- ~/CHEN_PENG_Projects/

- -->CHEN_PENG_Docs #各类文档

- -->clusterWebGUI/ #flask，上传界面

- -->clusterWebResult/ #flask，结果界面

- -->spark-streaming-kafka-0-8-assembly_2.11-2.3.1.jar #spark需要的jar包

- -->run_clusterZSL.sh #提交spark任务的脚本

---

**Project2: Distributed XGBoost on Spark**

~/CHEN_PENG_Projects/

- --> XGBoost/ #Scala+Spark xgboost项目（IntelliJ IDEA）

- --> run.sh #提交整个xgboost训练项目到spark

- --> distributed_workshop/

- --> data/ #存放所有数据

- --> fileBank/ #存放倒排用到的数据

- --> full_chunking.py #分词

- --> new_read_all.py #读取日语word embedding

- --> data_transform.py #生成xgboost training数据

- --> nn_data_transform.py #生成cnn training数据

- --> inverted_indexer.py #倒排pyspark文件

- --> run_inverted_index.sh #运行pyspark倒排脚本

- --> .....

- --> NN/ #Pytorch CNN项目
