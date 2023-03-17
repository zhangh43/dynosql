# dynosql

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
monograph_aws_access_key_id， monograph_aws_secret_key， monograph_dynamodb_region

```
[mariadb]
plugin_maturity=experimental
datadir=/home/ubuntu/data0
max_connections=5000
skip-log-bin
port=3306
socket=/tmp/mysqld3306.sock
plugin_load_add=ha_monograph
monograph
monograph_core_num=1
monograph_local_ip=127.0.0.1:8000
monograph_ip_list=127.0.0.1:8000
#monograph_metrics_port=18081
monograph_checkpointer_interval_sec=86400
monograph_node_memory_limit_mb=1600
monograph_kv_storage=dynamo
monograph_enable_mvcc=off
monograph_checkpointer_delay_sec=0
#skip-grant-tables

monograph_keyspace_name=mono
monograph_aws_access_key_id=your id
monograph_aws_secret_key=your key
monograph_dynamodb_region=your region
```

## 初始化数据库
数据库安装目录 /home/ubuntu/install

注：初始化时间比较长，目前简单添加8秒等待确保DynamoDB创建表成为CREATED状态，后续会优化。

```
/home/ubuntu/install/scripts/mysql_install_db --defaults-file=/home/ubuntu/dynosql.cnf --basedir=/home/ubuntu/install --datadir=/home/ubuntu/data0 --plugin-dir=/home/ubuntu/install/lib/plugin
```

初始化完毕后，Dynamo会出现很多系统表。

## 启动数据库

```
export LD_LIBRARY_PATH=/workspace/mariadb/install/lib
sudo mkdir -p /workspace/mariadb/install
sudo chown -R ubuntu:ubuntu /workspace
ln -s /home/ubuntu/install /workspace/mariadb/install
/home/ubuntu/install/bin/mysqld --defaults-file=/home/ubuntu/dynosql.cnf > mysql_log 2>&1 &
```

## 查询数据库

```
/home/ubuntu/install/bin/mysql -u ubuntu -S /tmp/mysqld3306.sock
```

## 

创建表和加载少量数据，表默认是按需计费模式。
```
sysbench /usr/share/sysbench/oltp_common.lua --mysql_storage_engine=monograph --tables=1 --table_size=1000 --mysql-user=ubuntu --mysql-socket=/tmp/mysqld3306.sock --mysql-db=test --time=600 --threads=100 --report-interval=5 --auto_inc=off --create_secondary=false prepare
```

修改mono12.t._test_sbtest1的RCU和WCU，默认是按需模式。注：如果WCU过低，OLTP_INSERT的时候会报错，mysql log report：`Dynamo put error:The level of configured provisioned throughput for the table was exceeded. Consider increasing your provisioning level with the UpdateTable API.`

OLTP_POINT_SELECT 查询

```
sysbench /usr/share/sysbench/oltp_point_select.lua --mysql_storage_engine=monograph --tables=1 --table_size=1000 --mysql-user=ubuntu --mysql-socket=/tmp/mysqld3306.sock --mysql-db=test --time=60 --threads=100 --report-interval=5 --auto_inc=off --create_secondary=false run
```

OLTP_INSERT 查询

```
sysbench /usr/share/sysbench/oltp_insert.lua --mysql_storage_engine=monograph --tables=1 --table_size=1000 --mysql-user=ubuntu --mysql-socket=/tmp/mysqld3306.sock --mysql-db=test --time=60 --threads=100 --report-interval=5 --auto_inc=off --create_secondary=false run
```

性能目前还在优化中
