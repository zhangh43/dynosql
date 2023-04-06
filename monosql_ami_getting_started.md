# MonoSQL(AMI based) Getting Started

## Get MonoSQL AMI

AMI name: MonoSQLServer

AMI includes MonoSQL server. Auto scaling group uses this AMI to create instances.

AMI name: MonoSQLMonitor

AMI includes Prometheus and Grafana. Create a standalone instance with this AMI. It will collect MonoSQL server metrics from Auto scaling group named `MonoSQL`.

TODO: In future, we integrate AMI with AWS Market Place. User can buy VM with MonoSQL AMI through Market Place.




## Bootstrap MonoSQL

MonoSQL needs run bootstrap to create system table on DynamoDB.

Create a vm using MonoSQL AMI and bootstrap DynamoDB with commands below

```
/home/ubuntu/install/scripts/mysql_install_db --defaults-file=/home/ubuntu/dynosql.cnf --basedir=/home/ubuntu/install --datadir=/home/ubuntu/data0 --plugin-dir=/home/ubuntu/install/lib/plugin > log 2>&1 &
```

Create User `sysb` to run sysbench benchmark later

```
# startdb
/home/ubuntu/install/bin/mysqld --defaults-file=/home/ubuntu/dynosql.cnf > mysql_log 2>&1 &

# bin/mysql -uubuntu -S /tmp/mysqld3306.sock
delete from mysql.user where User='';
CREATE USER 'sysb'@'%' IDENTIFIED BY 'sysb';
GRANT ALL PRIVILEGES ON * . * TO  'sysb'@'%';
FLUSH PRIVILEGES;
```


## Create Auto Scaling Group

Create auto-scaling group with MonoSQL AMI.

Help link: https://docs.aws.amazon.com/autoscaling/ec2/userguide/get-started-with-ec2-auto-scaling.html?icmpid=docs_ec2as_help_panel

Max vm number: 10
Expected vm number: 3
Min vm number: 0

The security group should allow TCP trafic from port 3306.

## Create Network Load Balancer

Create auto-scaling group with MonoSQL AMI.

Help link: https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html

Attach the auto-scaling group to NLB.

Choose NLB type to internal (We use the client vm in the same subnet to connect to the NLB server)

Set listener port of NLB to 3306.

## Test Using Sysbench

Suppose the NLB DNS is `MonoSQL-1-56977fbaf2a27fef.elb.ap-northeast-1.amazonaws.com`

```
sysbench /usr/share/sysbench/oltp_insert.lua --mysql_storage_engine=monograph --tables=1 --table_size=100000 
--mysql-user=sysb --mysql-host=MonoSQL-1-56977fbaf2a27fef.elb.ap-northeast-1.amazonaws.com --mysql-port=3306 
--mysql-password=sysb --mysql-db=test --time=120 --threads=100 --report-interval=5 --auto_inc=off 
--create_secondary=false --mysql-ignore-errors=all prepare

sysbench /usr/share/sysbench/oltp_insert.lua --mysql_storage_engine=monograph --tables=1 --table_size=100000 
--mysql-user=sysb --mysql-host=MonoSQL-1-56977fbaf2a27fef.elb.ap-northeast-1.amazonaws.com --mysql-port=3306 
--mysql-password=sysb --mysql-db=test --time=120 --threads=300 --report-interval=5 --auto_inc=off 
--create_secondary=false --mysql-ignore-errors=all run
```
