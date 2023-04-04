# MonoSQL (SQL wrapper for DynamoDB)

## 安装依赖
libgoogle-glog-dev
openjdk-11-jre-headless
less
python3
libsnappy-dev
libuv1-dev
libgflags-dev
libprotobuf-dev
libleveldb-dev
wget
curl
unzip
sysbench

## 解压数据库
tar -zxvf monographdb-release-bin.tar.gz

## 配置数据库
请在以下配置文件/home/ubuntu/dynosql.cnf中填写DynamoDB配置信息：

```
[mariadb]
plugin_maturity=experimental
datadir=/home/ubuntu/data0
max_connections=5000
skip-log-bin
port=3306
socket=/tmp/mysqld3306.sock
plugin_dir=/home/ubuntu/install/lib/plugin
plugin_load_add=ha_monograph
monograph
monograph_kv_storage=dynamo
monograph_dynamodb_default_credentials=on
monograph_dynamodb_region=ap-northeast-1
monograph_keyspace_name=mono
```

## 初始化数据库
数据库安装目录 /home/ubuntu/install

```
export LD_LIBRARY_PATH=/workspace/mariadb/install/lib
/home/ubuntu/install/scripts/mysql_install_db --defaults-file=/home/ubuntu/dynosql.cnf --basedir=/home/ubuntu/install --datadir=/home/ubuntu/data0 --plugin-dir=/home/ubuntu/install/lib/plugin
```

初始化完毕后，Dynamo会出现很多系统表。

## 启动数据库

```
/home/ubuntu/install/bin/mysqld --defaults-file=/home/ubuntu/dynosql.cnf > mysql_log 2>&1 &
```

## 查询数据库

```
/home/ubuntu/install/bin/mysql -u ubuntu -S /tmp/mysqld3306.sock
```


## 创建用户

```
# bin/mysql -uubuntu -S /tmp/mysqld3306.sock
delete from mysql.user where User='';
CREATE USER 'sysb'@'%' IDENTIFIED BY 'sysb';
GRANT ALL PRIVILEGES ON * . * TO  'sysb'@'%';
FLUSH PRIVILEGES;
```

## Benchmark

### Ycsb


创建表
```
# bin/mysql -u sysb -h 127.0.0.1 --port=3306 -p d3
create database d3;
CREATE TABLE usertable (
        YCSB_KEY VARCHAR(255) PRIMARY KEY,
        FIELD0 TEXT, FIELD1 TEXT,
        FIELD2 TEXT, FIELD3 TEXT,
        FIELD4 TEXT, FIELD5 TEXT,
        FIELD6 TEXT, FIELD7 TEXT,
        FIELD8 TEXT, FIELD9 TEXT
);
```

灌入数据
```
./bin/ycsb load jdbc -P workloads/workloada -P jdbc-binding/conf/db.properties -cp /usr/share/java/mariadb-java-client-2.5.3.jar -p fieldcount=2 -p recordcount=1000 -threads 20
```


workloads/workloadc是只读查询 readproportion=1
workloads/workloadi是只写查询 insertproportion=1

DynoSQL Result

```
./bin/ycsb run jdbc -P workloads/workloadc -P jdbc-binding/conf/db.properties -cp /usr/share/java/mariadb-java-client-2.5.3.jar -p fieldcount=2 -p recordcount=1000 -p operationcount=400000 -threads 100

[READ], Operations, 400000
[READ], AverageLatency(us), 4990.82621
[READ], MinLatency(us), 2360
[READ], MaxLatency(us), 287743
[READ], 95thPercentileLatency(us), 8223
[READ], 99thPercentileLatency(us), 11671
[READ], Return=OK, 400000

./bin/ycsb run jdbc -P workloads/workloadi -P jdbc-binding/conf/db.properties -cp /usr/share/java/mariadb-java-client-2.5.3.jar -p fieldcount=2 -p recordcount=1000 -p operationcount=400000 -threads 100
[INSERT], Operations, 400000
[INSERT], AverageLatency(us), 6820.44114
[INSERT], MinLatency(us), 4380
[INSERT], MaxLatency(us), 236031
[INSERT], 95thPercentileLatency(us), 9487
```

DynamoDB Result

```
./bin/ycsb run dynamodb -P workloads/workloadc -P dynamodb-binding/conf/dynamodb.properties -p fieldcount=2 -p recordcount=1000 -p operationcount=400000 -threads 100
[READ], Operations, 400000
[READ], AverageLatency(us), 3514.1403425
[READ], MinLatency(us), 2644
[READ], MaxLatency(us), 278527
[READ], 95thPercentileLatency(us), 4175
[READ], 99thPercentileLatency(us), 9383
[READ], Return=OK, 400000

./bin/ycsb run dynamodb -P workloads/workloadi -P dynamodb-binding/conf/dynamodb.properties -p fieldcount=2 -p recordcount=1000 -p operationcount=400000 -threads 100
[INSERT], Operations, 400000
[INSERT], AverageLatency(us), 6333.343
[INSERT], MinLatency(us), 4592
[INSERT], MaxLatency(us), 305919
[INSERT], 95thPercentileLatency(us), 7751
[INSERT], 99thPercentileLatency(us), 14687
[INSERT], Return=OK, 400000
```

### Sysbench

创建表和加载少量数据，表默认是按需计费模式

```
sysbench /usr/share/sysbench/oltp_common.lua --mysql_storage_engine=monograph --tables=1 --table_size=1000 --mysql-user=ubuntu --mysql-socket=/tmp/mysqld3306.sock --mysql-db=test --time=600 --threads=100 --report-interval=5 --auto_inc=off --create_secondary=false prepare
```

修改mono12.t._test_sbtest1的RCU和WCU，默认是按需模式。注：如果WCU过低，OLTP_INSERT的时候会报错，mysql log report：`Dynamo put error:The level of configured provisioned throughput for the table was exceeded. Consider increasing your provisioning level with the UpdateTable API.`

#### OLTP_POINT_SELECT 查询

单节点

```
sysbench /usr/share/sysbench/oltp_point_select.lua --mysql_storage_engine=monograph --tables=1 --table_size=100000 --mysql-user=sysb --mysql-host=172.31.44.86 --mysql-port=3306 --mysql-password=sysb --mysql-db=test --time=120 --threads=100 --report-interval=5 --auto_inc=off --create_secondary=false --mysql-ignore-errors=all run
```

性能结果

```
transactions:                        (22745.55 per sec.)
Latency (ms):
         min:                                    2.03
         avg:                                    4.40
         max:                                  225.42
         95th percentile:                        7.70
```

配置haproxy，测试两个节点,性能结果表明线性可扩展

```
sysbench /usr/share/sysbench/oltp_point_select.lua --mysql_storage_engine=monograph --tables=1 --table_size=100000 --mysql-user=sysb --mysql-host=127.0.0.1 --mysql-port=3390 --mysql-password=sysb --mysql-db=test --time=120 --threads=200 --report-interval=5 --auto_inc=off --create_secondary=false --mysql-ignore-errors=all run
```

性能结果

```
transactions:                      (43990.35 per sec.)
Latency (ms):
         min:                                    1.98
         avg:                                    4.54
         max:                                 1092.03
         95th percentile:                        7.56
```

#### OLTP_INSERT 查询

创建空表，禁用auto_increment和二级索引

```
sysbench /usr/share/sysbench/oltp_insert.lua --mysql_storage_engine=monograph --tables=1 --table_size=100000 --mysql-user=sysb --mysql-host=127.0.0.1 --mysql-port=3390 --mysql-password=sysb --mysql-db=test --time=120 --threads=100 --report-interval=5 --auto_inc=off --create_secondary=false --mysql-ignore-errors=all prepare
```

修改RCU和WCU，等待一段时间。

```
sysbench /usr/share/sysbench/oltp_insert.lua --mysql_storage_engine=monograph --tables=1 --table_size=1000 --mysql-user=sysb --mysql-host=127.0.0.1 --mysql-port=3390 --mysql-password=sysb --mysql-db=test --time=120 --threads=100 --report-interval=5 --auto_inc=off --create_secondary=false --mysql-ignore-errors=all run
```

性能结果
```
Transactions 14768.40 per sec
Latency (ms):
         min:                                    4.16
         avg:                                    6.73
         max:                                 2306.97
         95th percentile:                        9.22
```

性价比分析：

SQL Wrapper c5.4xlarge vm price： $0.856/hour

Dynamo provision 价格： $0.00065 per WCU per hour * 14768 = $9.5992/hour

配置haproxy，测试两个节点,性能结果表明线性可扩展

```
sysbench /usr/share/sysbench/oltp_insert.lua --mysql_storage_engine=monograph --tables=1 --table_size=1000 --mysql-user=sysb --mysql-host=127.0.0.1 --mysql-port=3390 --mysql-password=sysb --mysql-db=test --time=120 --threads=200 --report-interval=5 --auto_inc=off --create_secondary=false run
```

```
Transactions: 29545.32 per sec.
Latency (ms):
         min:                                    4.09
         avg:                                    6.77
         max:                                 2370.54
         95th percentile:                        9.22
```

#### Update查询

```
sysbench /usr/share/sysbench/oltp_update_non_index.lua --mysql_storage_engine=monograph --tables=1 --table_size=1000 --mysql-user=sysb --mysql-host=172.31.44.86 --mysql-port=3306 --mysql-password=sysb --mysql-db=test --time=120 --threads=100 --report-interval=5 --auto_inc=off --create_secondary=false --mysql-ignore-errors=all run
```

Update_non_index 性能结果（单节点）
```
transactions:                        1227908 (10136.47 per sec.)
Latency (ms):
         min:                                    4.30
         avg:                                    9.78
         max:                                25724.58
         95th percentile:                       12.98
```
