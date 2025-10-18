#node 
![[Pasted image 20240304235054.png]]

## 需求分析
![[Pasted image 20240305000808.png]]
## 项目实现
### 业务数据
#### 模拟数据生产
![[Pasted image 20240305135211.png]]
```shell
# run_affairs_db_productor.sh 数据生产脚本内容
--------------------------------------------------
#!/bin/bash  
  
java -jar gmall2020-mock-db-2021-11-14.jar
--------------------------------------------------
```
#### 使用flinkcdc 同步mysql到hbase
在原本的处理方案中，使用maxwell去监控mysql的binlog到kafka中，然后再去做相关的处理，但是本质就是将业务数据表中的维度表同步到hbase中。所以，我这里就不做繁琐的操作，直接使用flink cdc做数据同步。

```json
# mysql binlog 读取操作：
{  
    "before": null,  
    "after": {  
        "id": 8,  
        "name": "kik",  
        "age": 13  
    },  
    "source": {  
        "version": "1.9.7.Final",  
        "connector": "mysql",  
        "name": "mysql_binlog_source",  
        "ts_ms": 0,  
        "snapshot": "false",  
        "db": "test",  
        "sequence": null,  
        "table": "users",  
        "server_id": 0,  
        "gtid": null,  
        "file": "",  
        "pos": 0,  
        "row": 0,  
        "thread": null,  
        "query": null  
    },  
    "op": "r",  
    "ts_ms": 1709662532886,  
    "transaction": null  
}

# 插入数据：
{  
    "before": null,  
    "after": {  
        "id": 9,  
        "name": "kk",  
        "age": 19  
    },  
    "source": {  
        "version": "1.9.7.Final",  
        "connector": "mysql",  
        "name": "mysql_binlog_source",  
        "ts_ms": 1709666522000,  
        "snapshot": "false",  
        "db": "test",  
        "sequence": null,  
        "table": "users",  
        "server_id": 1,  
        "gtid": null,  
        "file": "mysql-bin.000001",  
        "pos": 1206510,  
        "row": 0,  
        "thread": 294,  
        "query": null  
    },  
    "op": "c",  
    "ts_ms": 1709666522674,  
    "transaction": null  
}
# 删除操作
{  
    "before": {  
        "id": 11,  
        "name": "keaven",  
        "age": 20  
    },  
    "after": null,  
    "source": {  
        "version": "1.9.7.Final",  
        "connector": "mysql",  
        "name": "mysql_binlog_source",  
        "ts_ms": 1709666942000,  
        "snapshot": "false",  
        "db": "test",  
        "sequence": null,  
        "table": "users",  
        "server_id": 1,  
        "gtid": null,  
        "file": "mysql-bin.000001",  
        "pos": 1207110,  
        "row": 0,  
        "thread": 294,  
        "query": null  
    },  
    "op": "d",  
    "ts_ms": 1709666942447,  
    "transaction": null  
}

# 修改
{  
    "before": {  
        "id": 10,  
        "name": "kk",  
        "age": 19  
    },  
    "after": {  
        "id": 10,  
        "name": "bvane",  
        "age": 19  
    },  
    "source": {  
        "version": "1.9.7.Final",  
        "connector": "mysql",  
        "name": "mysql_binlog_source",  
        "ts_ms": 1709667065000,  
        "snapshot": "false",  
        "db": "test",  
        "sequence": null,  
        "table": "users",  
        "server_id": 1,  
        "gtid": null,  
        "file": "mysql-bin.000001",  
        "pos": 1207415,  
        "row": 0,  
        "thread": 294,  
        "query": null  
    },  
    "op": "u",  
    "ts_ms": 1709667065304,  
    "transaction": null  
}
```

```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    age INT
);

UPDATE users
SET  name = 'Alices'
WHERE id = 2;

```
```shell

gmall\.activity_info,gmall\.activity_rule,gmall\.activity_sku,gmall\.base_category1,gmall\.base_category2,gmall\.base_category3,gmall\.base_province,gmall\.base_region,gmall\.base_trademark,gmall\.coupon_info,gmall\.coupon_range,gmall\.financial_sku_cost,gmall\.sku_info,gmall\.spu_info,gmall\.user_info

```

```sql

ALTER TABLE base_province ADD PRIMARY KEY (id);
ALTER TABLE base_region ADD PRIMARY KEY (id);



UPDATE user_info
SET birthday = DATE_FORMAT(birthday, '%Y-%m-%d');


ALTER TABLE user_info
ADD COLUMN date_string VARCHAR(10);


UPDATE user_info
SET date_string = DATE_FORMAT(birthday, '%Y-%m-%d');

ALTER TABLE user_info
DROP COLUMN birthday,
CHANGE COLUMN date_string birthday VARCHAR(10);

```
```sql

EXECUTE CDCSOURCE mysql2mongo WITH (
'connector' = 'mysql-cdc',
'hostname' = '127.0.0.1',
'port' = '3306',
'username' = 'root',
'password' = 'kone123456',
'checkpoint' = '10000',
'scan.startup.mode' = 'initial',
'parallelism' = '1',
'table-name' = 'test\.users',
'sink.connector' = 'mongodb',
'sink.uri' = 'mongodb://localhost:27017',
'sink.database' = 'test',
'collection' = 'users'
);


org.apache.flink.table.api.ValidationException: Unable to create a sink for writing table
```
### 日志数据
#### 模拟数据生产
![[Pasted image 20240305004230.png]]
```shell
# 在run_logs_file_productor中
--------------------------------------------------
#!/bin/bash  
  
num=$1  
echo $num  
for ((i=1; i<=$1; i++))  
do  
  echo "================= ${i} ======================"  java -jar gmall2020-mock-log-2021-10-10.jar  
done
--------------------------------------------------
```
在上面的需求分析中，我们是将数据放入到日志文件applogs中，然后使用flume打到kafka中。但是这个步骤，我们省略，直接将其打到kafka上
```shell
# kafka相关配置
mock:  
  kafka-server: "localhost:9092"  
  kafka-topic: "gmall_ods_base_logs"
```
##### 测试kafka
```shell
# 生产者进行生产（logs-file-data-productor目录下）
run_logs_file_productor.sh
# 消费者进行消费
kafka-console-consumer.sh  --topic gmall_ods_base_logs --bootstrap-server localhost:9092 --group gmall_group_base_logs
```
![[Pasted image 20240305133936.png]]
表示成功

```scala
ExecutionEnvironment.getExecutionEnvironment
```