### flink on eks workshop

```markdown
* Flink CDC DataStream API解析MySQL Binlog发送到Kafka，支持按库发送到不同Topic, 也可以发送到同一个Topic
* 自定义FlinkKafkaPartitioner, 数据库名，表名，主键值三个拼接作为partition key, 保证相同主键的记录发送到Kafka的同一个分区，保证消息顺序。
* Flink CDC支持增量快照算法，全局无锁，Batch阶段的checkpoint, 但需要表有主键，如果没有主键列增量快照算法就不可用，无法同步数据，需要设置scan.incremental.snapshot.enabled=false禁用增量快照
* 支持MySQL指定从binlog位置或者binlog时间点解析数据
* 加入-chunk_size参数,默认值8096,全量阶段如果表比较大,表的单行数据比较大,产生OOM时可以调小该值
* 增加Debezium Custom Converter处理Datetime类型转换和时区问题
* -table_pk 参数中指定column_max_length参数,col1=10|col2=10,表示col1列保留10个字符,col2列保留10个字符,多个列以竖线分割,注意-table_pk的json参数需要用反斜杠转义,例子如下
  '[{\"db\":\"test_db\",\"table\":\"product\",\"primary_key\":\"id\",\"column_max_length\":\"col1=10|col2=10\"}]'
```


#### 使用方式
```shell
# Main Class : MySQLCDC2AWSMSK
# 本地调试参数
-project_env local or prod # local表示本地运行，prod表示KDA运行
-disable_chaining false or true # 是否禁用flink operator chaining 
-delivery_guarantee at_least_once # kafka的投递语义at_least_once or exactly_once,建议at_least_once
-host localhost:3306 # mysql 地址
-username xxx # mysql 用户名
-password xxx # mysql 密码
-db_list test_db # 需要同步的数据库，支持正则，多个可以逗号分隔
-tb_list test_db.product.*,test_db.product  # 需要同步的表 支持正则，多个可以逗号分隔
-server_id 10000-10010 # 在快照读取之前，Source 不需要数据库锁权限。如果希望 Source 并行运行，则每个并行 Readers 都应该具有唯一的 Server id，所以 Server id 必须是类似 `5400-6400` 的范围，并且该范围必须大于并行度。
-server_time_zone Etc/GMT # mysql 时区
-position latest or initial or mysql-bin.000003 or mysql-bin.000003:123 or gtid:24DA167-0C0C-11E8-8442-00059A3C7B00:1-19 or timestamp:1667232000000 # latest从当前CDC开始同步，initial先快照再CDC, binlog_file_name 指定binlog文件, binlog_file_name:position 指定binlog文件和位置,gtid:xxx 指定gtid, timestamp:13位时间戳 指定时间戳
-kafka_broker localhost:9092 # kafka 地址
-topic test-cdc-1 # topic 名称, 如果所有的数据都发送到同一个topic,设定要发送的topic名称
-topic_prefix flink_cdc_ # 如果按照数据库划分topic,不同的数据库中表发送到不同topic,可以设定topic前缀，topic名称会被设定为 前缀+数据库名。 设定了-topic_prefix参数后，-topic参数不再生效
-table_pk [{"db":"test_db","table":"product","primary_key":"pid"},{"db":"test_db","table":"product_01","primary_key":"pid"}] # 需要同步的表的主键
# max.request.size 默认1MB,这里设置的10MB
-kafka_properties 'max.request.size=1073741824,xxxx=xxxx' # kafka生产者参数,多个以逗号分隔
-chunk_size 8090 # 默认值8096，全量阶段如果表比较大，表的单行数据比较大，产生OOM时，可以调小该值
```

#### build
```sh
mvn clean package -Dscope.type=provided
#编译好的JAR
https://dxs9dnjebzm6y.cloudfront.net/tmp/flink-cdc-msk-1.0-SNAPSHOT-202404100935.jar
```
   
#### EMR on EKS
* EMR 6.15.0 Flink 1.17.1
##### flink operator
```yaml
apiVersion: flink.apache.org/v1beta1
kind: FlinkDeployment
metadata:
  name: flink-cdc-operator
spec:
  flinkConfiguration:
    taskmanager.numberOfTaskSlots: "2"
    state.checkpoints.dir: $CHECKPOINT_S3_STORAGE_PATH
    state.savepoints.dir: $SAVEPOINT_S3_STORAGE_PATH 
  flinkVersion: v1_17
  executionRoleArn: $EMR_EXECUTION_ROLE_ARN
  emrReleaseLabel: "emr-6.15.0-flink-latest"
  jobManager:
    storageDir: $HIGH_AVAILABILITY_STORAGE_PATH
    highAvailabilityEnabled: true
    resource:
      memory: "2048m"
      cpu: 1
  taskManager:
    resource:
      memory: "2048m"
      cpu: 1
  job:
    jarURI: $FLINK_CDC_JOB_JAR
    entryClass:  com.aws.analytics.MySQLCDC2AWSMSK
    args:
      - "-project_env"
      - "prod"
      - "-disable_chaining"
      - "true"
      - "-delivery_guarantee"
      - "at_least_once"
      - "-host"
      - "$MYSQL_HOST:3306"
      - "-username"
      - "$MYSQL_USER"
      - "-password"
      - "$MYSQL_PWD"
      - "-db_list"
      - "sbtest"
      - "-tb_list"
      - "sbtest.sbtest.*"
      - "-server_id"
      - "200200-200300"
      - "-server_time_zone"
      - "US/Eastern"
      - "-kafka_broker"
      - "$MSK_BROKER"
      - "-topic"
      - "${CDC_TOPIC_NAME}"
      - "-table_pk"
      - "[{\"db\":\"sbtest\",\"table\":\"sbtest1\",\"primary_key\":\"id\"}]"
      - "-checkpoint_interval"
      - "30"
      - "-checkpoint_dir"
      - "s3://${BUCKET_NAME}/flink/checkpoint/test/"
      - "-parallel"
      - "4"
      - "-kafka_properties"
      - "max.request.size=1073741824"
      - "-chunk_size"
      - "8090"
    parallelism: 2
    upgradeMode: savepoint
    savepointTriggerNonce: 0
  monitoringConfiguration:
    cloudWatchMonitoringConfiguration:
       logGroupName: $LOG_GROUP_NAME
```
#### Dockerfile
```shell
cat <<EOF > Dockerfile
ARG EMR_VERSION
FROM public.ecr.aws/emr-on-eks/flink/emr-\${EMR_VERSION}-flink:latest
USER root
ARG FLINK_VERSION="1.17.1"
ENV FLINK_HOME="/usr/lib/flink/"
ENV FLINK_VERSION=\${FLINK_VERSION}
ENV MAVEN_VERSION="3.9.6"
ENV MAVEN_URL="https://apache.osuosl.org/maven/maven-3/"\${MAVEN_VERSION}"/binaries"

RUN mkdir -p /usr/share/maven
RUN curl -o /tmp/apache-maven.tar.gz \${MAVEN_URL}/apache-maven-\${MAVEN_VERSION}-bin.tar.gz && \
    tar -xzf /tmp/apache-maven.tar.gz -C /usr/share/maven --strip-components=1 && \
    rm -f /tmp/apache-maven.tar.gz && \
    ln -s /usr/share/maven/bin/mvn /usr/bin/mvn

RUN mkdir -p \$FLINK_HOME/usrlib
RUN rm -rf /usr/bin/python && ln -s /usr/bin/python2 /usr/bin/python
RUN yum -y install git && git clone https://github.com/yhyyz/flink-on-eks-workshop.git /tmp/flink-on-eks-workshop && \
    cd /tmp/flink-on-eks-workshop/ && \
    git pull && \
    cd /tmp/flink-on-eks-workshop/flink-cdc-msk && \
    mvn clean package -Dscope.type=provided

RUN rm -rf /usr/bin/python && ln -s /usr/bin/python3 /usr/bin/python

RUN cp /tmp/flink-on-eks-workshop/flink-cdc-msk/target/flink-cdc-msk-*.jar \$FLINK_HOME/usrlib/

# Use hadoop user and group 
USER hadoop:hadoop
EOF
```