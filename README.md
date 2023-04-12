# MonoSQL (SQL Wrapper for DynamoDB)

## Resources

1. MonoSQL is recommended to use AWS Network Load Balancer and Auto Scaling Group to deploy.

    Refer to Getting Startted Guild: https://github.com/zhangh43/dynosql/blob/main/monosql_ami_getting_started.md


2. Performance Benchmark is tested against Sysbench and YCSB.
    
    Refer to https://github.com/zhangh43/dynosql/blob/main/monosql_benchmark.md

3. Deploy MonoSQL through tarball is not recommended, but you can also check how to do it at https://github.com/zhangh43/dynosql/blob/main/monosql_benchmark.md

## Introduction to MonoSQL

MonoSQL is the SQL wrapper for DynamoDB. It helps the users from Mysql/MariaDB to migrate to DynamoDB with less effort to modify their application and remaining to use JDBC/ODBC to connect to you data.

MonoSQL server is stateless, all the catalog and data are stored in DynamoDB. SQL queries will be parsed, optimized and executed by MonoSQL server, while MonoSQL server will ask DynamoDB for the real data by using GetItem, PutItem etc. request.

MonoSQL can be deployed using AWS AutoScaling Group to achieve horizontal scale and automatically expand and shrink based on workload by integrating with cloud watch (Coming Soon).


The feature of MonoSQL includes:
1. User & Privilege management. Use `CREATE USER` command manage user in MonoSQL. Use `GRANT` command to manage privilege. Both are MySQL compatible.
2. DDL operation. Use `CREATE TABLE` and `DROP TABLE` to manage table schema.
3. DML operation. `SELECT`, `INSERT`, `UPDATE`, `DELETE` commands are supported.
4. Join operator.
5. Aggregation operator.
6. CTE and Recursive CTE operator.
7. And more MySQL features

Not supported feature currently:
1. `CREATE INDEX` is not supported (Coming Soon!)
2. `FLUSH PRIVILEGE` applies on all the nodes is not supported (Coming Soon!)
3. `ALTER TABLE` is not supported.
4. `Trigger` is not supported.


## Best Practice

1. Don't use full table scan in MonoSQL since it will cost a lot of RCU on DynamoDB. Currently you will receive an error when running a full table scan. Turn on session variable `full_tbl_scan` to force the scan query to be executed.

2. Limit the number of your primary key less than `two`. One is for partition key and the other is for sort key(optional) in DynamoDB. It will avoid packing the multi columns into a single filed as sort key since packing will lose the readablity in native DynamoDB and introduce limitation when creating index in future.

3. No statistics is stored in MonoSQL hence you can use optimizer hint such as `use index index_name` if the plan is not desired. Refer to https://dev.mysql.com/doc/refman/8.0/en/index-hints.html.
