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
请在配置文件/home/ubuntu/dynosql.cnf中填写 monograph_aws_access_key_id monograph_aws_secret_key monograph_dynamodb_region

```
[mariadb]
plugin_maturity=experimental
datadir=/home/ubuntu/data0
datadir=/home/ubuntu/data0
max_connections=5000
skip-log-bin
port=3306
socket=/tmp/mysqld3306.sock
plugin_load_add=ha_monograph
monograph
monograph_cass_hosts=127.0.0.1
monograph_cass_user=cassandra
monograph_cass_password=cassandra
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

```

/home/ubuntu/install/scripts/mysql_install_db --defaults-file=/home/ubuntu/dynosql.cnf --basedir=/home/ubuntu/install --datadir=/home/ubuntu/data0 --plugin-dir=/home/ubuntu/install/lib/plugin
```

## 启动数据库

```
export LD_LIBRARY_PATH=/workspace/mariadb/install/lib
/home/ubuntu/install/bin/mysqld --defaults-file=/home/ubuntu/dynosql.cnf > mysql_log 2>&1 &
```

## 查询数据库

```
/home/ubuntu/install/bin/mysql -u ubuntu -S /tmp/mysqld3306.sock
```
