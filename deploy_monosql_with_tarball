# Deploy MonoSQL with Binary Tarball

## OS version

Ubuntu 20.04 (Other OS version will be supported soon)

## Install dependency

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

## Unzip tarball

```
# download monographdb-release-bin.tar.gz and place it into dir /home/ubuntu/
cd /home/ubuntu/
tar -zxvf monographdb-release-bin.tar.gz
# the binary is installed at path:
/home/ubuntu/install
```


## Set config file

create file /home/ubuntu/dynosql.cnf

```
[mariadb]
plugin_maturity=experimental
datadir=/home/ubuntu/data0
max_connections=1000
thread_handling=pool-of-threads
thread_pool_size=100
performance_schema=ON
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

performance-schema-instrument='stage/%=ON'
performance-schema-consumer-events-stages-current=ON
performance-schema-consumer-events-stages-history=ON
performance-schema-consumer-events-stages-history-long=ON

performance-schema-instrument='statement/%=ON'
performance-schema-consumer-events-statements-current=ON
performance-schema-consumer-events-statements-history=ON
performance-schema-consumer-events-statements-history-long=ON
performance-schema-consumer-statements-digest=ON

performance-schema-instrument='transaction=ON'
performance-schema-consumer-events-transactions-current=ON
performance-schema-consumer-events-transactions-history=ON
performance-schema-consumer-events-transactions-history-long=ON
```

## Bootstrap database

```
export LD_LIBRARY_PATH=/workspace/mariadb/install/lib
/home/ubuntu/install/scripts/mysql_install_db --defaults-file=/home/ubuntu/dynosql.cnf --basedir=/home/ubuntu/install --datadir=/home/ubuntu/data0 --plugin-dir=/home/ubuntu/install/lib/plugin
```

## Start database

```
/home/ubuntu/install/bin/mysqld --defaults-file=/home/ubuntu/dynosql.cnf > mysql_log 2>&1 &
```

## Connect database

```
/home/ubuntu/install/bin/mysql -u ubuntu -S /tmp/mysqld3306.sock
```
