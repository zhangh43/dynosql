# MonoSQL Benchmark Result

## Prerequisite

Follow document https://github.com/zhangh43/dynosql/blob/main/monosql_ami_getting_started.md to bootstrap cluster and create benchmark dbuser `sysb`.


## Sysbench benchmark

Test three sysbench workloads: OLTP_POINT_SELECT, INSERT_ONLY and UPDATE_NON_INDEX, which are corresponding to GetItem, PutItem and UpdateItem request in DynamoDB. 

### SELECT workload

1. Create table and load data with oltp_common.lua. Please disable auto_incement and secondary index. Note that the price mode of created table in DynamoDB is `on-demand`. You need to set a provsioned WCU and RCU to avoid throttle of DynamoDB. 

```
# Suppose the NLB DNS is MonoSQL-1-56977fbaf2a27fef.elb.ap-northeast-1.amazonaws.com

sysbench /usr/share/sysbench/oltp_common.lua --mysql_storage_engine=monograph --tables=1 --table_size=1000 --mysql-user=sysb --mysql-host=MonoSQL-1-56977fbaf2a27fef.elb.ap-northeast-1.amazonaws.com --mysql-port=3306 --mysql-password=sysb --mysql-db=test --time=120 --threads=100 --report-interval=5 --auto_inc=off --create_secondary=false --mysql-ignore-errors=all prepare
```


2. Run single node test by setting vm number in auto-scaling group to `1`.

```
sysbench /usr/share/sysbench/oltp_point_select.lua --mysql_storage_engine=monograph --tables=1 --table_size=1000 --mysql-user=sysb --mysql-host=MonoSQL-1-56977fbaf2a27fef.elb.ap-northeast-1.amazonaws.com --mysql-port=3306 --mysql-password=sysb --mysql-db=test --time=120 --threads=100 --report-interval=5 --auto_inc=off --create_secondary=false --mysql-ignore-errors=all run
```

Performance Result:

```
transactions:                        (22745.55 per sec.)
Latency (ms):
         min:                                    2.03
         avg:                                    4.40
         max:                                  225.42
         95th percentile:                        7.70
```

3. Run two nodes test by setting vm number in auto-scaling group to `2`.

```
sysbench /usr/share/sysbench/oltp_point_select.lua --mysql_storage_engine=monograph --tables=1 --table_size=1000 --mysql-user=sysb --mysql-host=MonoSQL-1-56977fbaf2a27fef.elb.ap-northeast-1.amazonaws.com --mysql-port=3306 --mysql-password=sysb --mysql-db=test --time=120 --threads=200 --report-interval=5 --auto_inc=off --create_secondary=false --mysql-ignore-errors=all run
```

Performance Result:

```
transactions:                      (43990.35 per sec.)
Latency (ms):
         min:                                    1.98
         avg:                                    4.54
         max:                                 1092.03
         95th percentile:                        7.56
```

The result shows linear scalability.

### INSERT workload

创建空表，禁用auto_increment和二级索引

1. Create table with oltp_insert.lua. Please disable auto_incement and secondary index. Note that the price mode of created table in DynamoDB is `on-demand`. You need to set a provsioned WCU and RCU to avoid throttle of DynamoDB. 

```
# Suppose the NLB DNS is MonoSQL-1-56977fbaf2a27fef.elb.ap-northeast-1.amazonaws.com

sysbench /usr/share/sysbench/oltp_insert.lua --mysql_storage_engine=monograph --tables=1 --table_size=1000 --mysql-user=sysb --mysql-host=MonoSQL-1-56977fbaf2a27fef.elb.ap-northeast-1.amazonaws.com --mysql-port=3306 --mysql-password=sysb --mysql-db=test --time=120 --threads=100 --report-interval=5 --auto_inc=off --create_secondary=false --mysql-ignore-errors=all prepare
```

2. Run single node test by setting vm number in auto-scaling group to `1`.

```
sysbench /usr/share/sysbench/oltp_insert.lua --mysql_storage_engine=monograph --tables=1 --table_size=1000 --mysql-user=sysb --mysql-host=MonoSQL-1-56977fbaf2a27fef.elb.ap-northeast-1.amazonaws.com --mysql-port=3306 --mysql-password=sysb --mysql-db=test --time=120 --threads=100 --report-interval=5 --auto_inc=off --create_secondary=false --mysql-ignore-errors=all run
```

Performance Result:

```
Transactions:                      (14768.40 per sec.)
Latency (ms):
         min:                                    4.16
         avg:                                    6.73
         max:                                 2306.97
         95th percentile:                        9.22
```



3. Run two nodes test by setting vm number in auto-scaling group to `2`.

```
sysbench /usr/share/sysbench/oltp_insert.lua --mysql_storage_engine=monograph --tables=1 --table_size=1000 --mysql-user=sysb --mysql-host=MonoSQL-1-56977fbaf2a27fef.elb.ap-northeast-1.amazonaws.com --mysql-port=3306 --mysql-password=sysb --mysql-db=test --time=120 --threads=200 --report-interval=5 --auto_inc=off --create_secondary=false --mysql-ignore-errors=all run
```

Performance Result:

```
Transactions:                      (29545.32 per sec.)
Latency (ms):
         min:                                    4.09
         avg:                                    6.77
         max:                                 2370.54
         95th percentile:                        9.22
```

#### Update Workload

1. Run single node test by setting vm number in auto-scaling group to `1`.

```
sysbench /usr/share/sysbench/oltp_update_non_index.lua --mysql_storage_engine=monograph --tables=1 --table_size=1000 --mysql-user=sysb --mysql-host=MonoSQL-1-56977fbaf2a27fef.elb.ap-northeast-1.amazonaws.com --mysql-port=3306 --mysql-password=sysb --mysql-db=test --time=120 --threads=100 --report-interval=5 --auto_inc=off --create_secondary=false --mysql-ignore-errors=all run
```

Performance Result:

```
transactions:                     (10136.47 per sec.)
Latency (ms):
         min:                                    4.30
         avg:                                    9.78
         max:                                25724.58
         95th percentile:                       12.98
```

## MonoSQL Cost Analysis

Based on the sysbench performance result, we can calcualte the additional cost when introducing MonoSQL.

Take OLTP_INSERT as example. One vm with type of `c5.4xlarge` can serve 14768 WCU.

The cost of DynamoDB with 14768 provisioned WCU is: `$9.5992/hour` (formula: $0.00065 per WCU per hour * 14768 = $9.5992/hour)

The additional cost of MonoSQL's vm cost is: `$0.856/hour` ( based on the price of c5.4xlarge)


## Ycsb Benchmark

1. Create Table

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

2. Load data

```
./bin/ycsb load jdbc -P workloads/workloada -P jdbc-binding/conf/db.properties -cp /usr/share/java/mariadb-java-client-2.5.3.jar -p fieldcount=2 -p recordcount=1000 -threads 20
```

3. Run test. Note that workloads/workloadc is ReadOnly workload with config readproportion=1 and workloads/workloadi is insert only workload with config insertproportion=1

MonoSQL Performance Result

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

Native DynamoDB Result

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
