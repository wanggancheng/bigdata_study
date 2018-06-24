``# 基本管理命令
## 启动命令


```shell
cd ${SPARK_HOME} && ./sbin/start-all.sh
```

## 使用基本命令示例



```
./bin/spark-submit --class org.apache.spark.examples.SparkPi --master spark://cdh01:7077 --executor-memory 2G --total-executor-cores 2 /home/appuser/app/spark/examples/jars/spark-examples_2.11-2.3.1.jar
```

