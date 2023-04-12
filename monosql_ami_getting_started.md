# MonoSQL(AMI based) Getting Started

## Prerequisite

1. AWS region must be `ap-northeast-1` currently.

2. Create AWS role `monosql` as the vm role when running MonoSQLServer and MonoSQL Monitor. Privileges of role `monosql`: `AmazonEC2ReadOnlyAccess`, `AmazonSQSFullAccess`, `AmazonDynamoDBFullAccess`, `CloudWatchLogsFullAccess`, `CloudWatchAgentServerPolicy`

## Get MonoSQL AMI

AMI name: `MonoSQLServer`

AMI includes MonoSQL server. Auto scaling group uses this AMI to create instances.

AMI name: `MonoSQLMonitor`

AMI includes Prometheus and Grafana. Create a standalone instance with this AMI. It will collect MonoSQL server metrics from Auto scaling group named `MonoSQL`.

TODO: In future, we integrate AMI with AWS Market Place. User can buy VM with MonoSQL AMI through Market Place.

## Bootstrap MonoSQL

MonoSQL needs run bootstrap to create system table on DynamoDB.

Create a vm using `MonoSQLServer` AMI and bootstrap DynamoDB with commands below

```
/home/ubuntu/install/scripts/mysql_install_db --defaults-file=/home/ubuntu/dynosql.cnf --basedir=/home/ubuntu/install --datadir=/home/ubuntu/data0 --plugin-dir=/home/ubuntu/install/lib/plugin > log 2>&1 &
```

`OK` in log file indicates the bootstrap succeeded.

Create user `sysb` to run sysbench benchmark and user `mono` for monitor.

```
# startdb.
/home/ubuntu/install/bin/mysqld --defaults-file=/home/ubuntu/dynosql.cnf > mysql_log 2>&1 &

# connect to db.
bin/mysql -uubuntu -S /tmp/mysqld3306.sock test

# create sysbench user and monitor user.
delete from mysql.user where User='';
CREATE USER 'sysb'@'%' IDENTIFIED BY 'sysb';
GRANT ALL PRIVILEGES ON * . * TO  'sysb'@'%';

CREATE USER IF NOT EXISTS 'mono'@'%' IDENTIFIED BY 'mono' WITH MAX_USER_CONNECTIONS 5;
GRANT ALL PRIVILEGES ON *.* TO 'mono'@'%' IDENTIFIED BY 'mono' WITH GRANT OPTION;
FLUSH PRIVILEGES;

```


## Create Auto Scaling Group

Create auto-scaling group with `MonoSQLServer` AMI.

Help link: https://docs.aws.amazon.com/autoscaling/ec2/userguide/get-started-with-ec2-auto-scaling.html?icmpid=docs_ec2as_help_panel

Group name: MonoSQL (group name is used by Prometheus and thus can not be changed)

Max vm number: 10

Expected vm number: 3

Min vm number: 0

The security group should allow TCP trafic from port 3306.

When creating auto-scaling group, the network load balancer could be created and attached at once. See section below.

## Create Network Load Balancer

Help link: https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html

Attach the auto-scaling group to NLB.

Choose NLB type to internal (We use the client vm in the same subnet to connect to the NLB server)

Set listener port of NLB to 3306.

## Create Prometheus and Grafana Node

Create Prometheus and Grafana with `MonoSQLMonitor` AMI.

Use web browser to access http://nodeip:3000 to get the metrics of MonoSQL server.

Note that the Prometheus will monitor auto-scaling group named `MonoSQL` automatically.

## Retrieving server logs using Cloud Watch

Server logs of vms in auto-scaling group are stored in Cloud Watch.

Log group name: monosql-service

Log name format: vmid-ip.region.compute.internal-monosql-service.log 

example: i-0b68393de9490367c-ip-172-31-39-136.ap-northeast-1.compute.internal-monosql-service.log

## Test Using Sysbench

Suppose the NLB DNS is `MonoSQL-1-56977fbaf2a27fef.elb.ap-northeast-1.amazonaws.com`

```
sysbench /usr/share/sysbench/oltp_insert.lua --mysql_storage_engine=monograph --tables=1 --table_size=100000 --mysql-user=sysb --mysql-host=MonoSQL-1-56977fbaf2a27fef.elb.ap-northeast-1.amazonaws.com --mysql-port=3306 --mysql-password=sysb --mysql-db=test --time=120 --threads=100 --report-interval=5 --auto_inc=off --create_secondary=false --mysql-ignore-errors=all prepare

sysbench /usr/share/sysbench/oltp_insert.lua --mysql_storage_engine=monograph --tables=1 --table_size=100000 --mysql-user=sysb --mysql-host=MonoSQL-1-56977fbaf2a27fef.elb.ap-northeast-1.amazonaws.com --mysql-port=3306 --mysql-password=sysb --mysql-db=test --time=120 --threads=300 --report-interval=5 --auto_inc=off --create_secondary=false --mysql-ignore-errors=all run
```
