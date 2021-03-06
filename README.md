tpch-kit
========

TPC-H benchmark kit with some modifications/additions

Official TPC-H benchmark - [http://www.tpc.org/tpch](http://www.tpc.org/tpch)

## Modifications

The following modifications have been added on top of the official TPC-H kit:

* modify `dbgen` to not print trailing delimiter
* add option for `dbgen` to output to stdout
* add compile support for macOS
* add define for PostgreSQL to support `LIMIT N` for `qgen`
* adjust `Makefile` defaults

## Setup

### Linux

Make sure the required development tools are installed:

Ubuntu:
```
sudo apt-get install git make gcc
```

CentOS/RHEL:
```
sudo yum install git make gcc
```

Then run the following commands to clone the repo and build the tools:

```
git clone https://github.com/gregrahn/tpch-kit.git
cd tpch-kit/dbgen
make MACHINE=LINUX DATABASE=SQLSERVER
```

### macOS

Make sure the required development tools are installed:

```
xcode-select --install
```

Then run the following commands to clone the repo and build the tools:

```
git clone https://github.com/gregrahn/tpch-kit.git
cd tpch-kit/dbgen
make MACHINE=MACOS DATABASE=SQLSERVER
```

## Using the TPC-H tools

### Generate databases
```
./dbgen -s 0.01

ls *.tbl

# if your mysql is started by docker: $ docker run --rm -d --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=1234 mysql:5.7
# docker exec -it mysql mysql -uroot -p1234
# ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '1234';

mysql -uroot -h 127.0.0.1 -P3306 -p1234  --local-infile < load_tbl_to_mysql.sql

# Dump db for later use
# mysqldump -uroot -p1234 -h 127.0.0.1 tpch > tpch.sql
```

### Run some SQL and check results

* https://github.com/catarinaribeir0/queries-tpch-dbgen-mysql

* http://www.tpc.org/tpc_documents_current_versions/pdf/tpc-h_v2.17.3.pdf

```sql
-- Q1 ##############################################################################
SELECT l_returnflag, 
       l_linestatus, 
       SUM(l_quantity)                                           AS sum_qty, 
       SUM(l_extendedprice)                                      AS 
       sum_base_price, 
       SUM(l_extendedprice * ( 1 - l_discount ))                 AS 
       sum_disc_price, 
       SUM(l_extendedprice * ( 1 - l_discount ) * ( 1 + l_tax )) AS sum_charge, 
       Avg(l_quantity)                                           AS avg_qty, 
       Avg(l_extendedprice)                                      AS avg_price, 
       Avg(l_discount)                                           AS avg_disc, 
       Count(*)                                                  AS count_order 
FROM   lineitem 
WHERE  l_shipdate <= DATE '1998-12-01' - interval '90' day 
GROUP  BY l_returnflag, 
          l_linestatus 
ORDER  BY l_returnflag, 
          l_linestatus; 

-- Q2 ################################################################################
SELECT s_acctbal, 
       s_name, 
       n_name, 
       p_partkey, 
       p_mfgr, 
       s_address, 
       s_phone, 
       s_comment 
FROM   part, 
       supplier, 
       partsupp, 
       nation, 
       region 
WHERE  p_partkey = ps_partkey 
       AND s_suppkey = ps_suppkey 
       AND p_size = 15 
       AND p_type LIKE '%BRASS' 
       AND s_nationkey = n_nationkey 
       AND n_regionkey = r_regionkey 
       AND r_name = 'EUROPE' 
       AND ps_supplycost = (SELECT Min(ps_supplycost) 
                            FROM   partsupp, 
                                   supplier, 
                                   nation, 
                                   region 
                            WHERE  p_partkey = ps_partkey 
                                   AND s_suppkey = ps_suppkey 
                                   AND s_nationkey = n_nationkey 
                                   AND n_regionkey = r_regionkey 
                                   AND r_name = 'EUROPE') 
ORDER  BY s_acctbal DESC, 
          n_name, 
          s_name, 
          p_partkey 
LIMIT  100; 
```


### Environment

Set these env variables correctly:

```
export DSS_CONFIG=/.../tpch-kit/dbgen
export DSS_QUERY=$DSS_CONFIG/queries
export DSS_PATH=/path-to-dir-for-output-files
```

### SQL dialect

See `Makefile` for the valid `DATABASE` values.  Details for each dialect can be found in `tpcd.h`.  Adjust the query templates in `tpch-kit/dbgen/queries` as need be.

### Data generation

Data generation is done via `dbgen`.  See `dbgen -h` for all options.  The environment variable `DSS_PATH` can be used to set the desired output location.

### Query generation

Query generation is done via `qgen`.  See `qgen -h` for all options.

The following command can be used to generate all 22 queries in numerical order for the 1GB scale factor (`-s 1`) using the default substitution variables (`-d`).

```
qgen -v -c -d -s 1 > tpch-stream.sql
```

To generate one query per file for SF 3000 (3TB) use:

```
for ((i=1;i<=22;i++)); do
  ./qgen -v -c -s 3000 ${i} > /tmp/sf3000/tpch-q${i}.sql
done
```
