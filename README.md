# Hadoop ETL UDFs

[![Build Status](https://travis-ci.org/EXASOL/hadoop-etl-udfs.svg?branch=master)](https://travis-ci.org/EXASOL/hadoop-etl-udfs)


###### Please note that this is an open source project which is officially supported by EXASOL. For any question, you can contact our support team.

## Table of Contents
1. [Overview](#overview)
2. [Deploying the Hadoop ETL UDFs](#deploying-the-hadoop-etl-udfs)
3. [Using the Hadoop ETL UDFs](#using-the-hadoop-etl-udfs)
4. [Building from Source](#building-from-source)
5. [Kerberos Authentication](#kerberos-authentication)
6. [Debugging](#debugging)


## Overview
Hadoop ETL UDFs are the main way to load data from Hadoop into EXASOL (HCatalog tables on HDFS).

The features in detail:
* Metadata are retrieved from HCatalog (HDFS files, file formats, columns, etc.).
* Supports all Hive SerDes (Parquet, ORC, RC, Avro, JSON, etc.).
* Supports compression for SerDe (e.g., ORC compression) and for Hive (```hive.exec.compress.*```).
* Supports complex data types (array, map, struct, union) and JsonPath. Values of complex data types are returned as JSON. You can also specify simple JSONPath expressions.
* Supports to specify filters which partitions to load.
* Parallel Transfer:
  * Data is loaded directly from the data node to one of the EXASOL nodes.
  * Parallelization is applied if the HCatalog table consists of multiple files.
  * Degree of parallelism can be controlled via an UDF parameter. The maximum degree is determined by the number of HDFS files and the number of EXASOL nodes and cores.


## Deploying the Hadoop ETL UDFs

Prerequisites:
* EXASOL Advanced Edition (version 6.0 or newer) or Free Small Business Edition.
* JDK & Maven to build from source
* Connectivity from EXASOL to Hadoop: Make sure that following Hadoop services can be accessed from EXASOL. In case of problems please use an [UDF to check the connectivity](https://www.exasol.com/support/browse/SOL-307).
  * Hive Metastore or WebHCatalog: All EXASOL nodes need access to either the Hive Metastore Server (recommended) or to WebHCatalog.
    * The Hive Metastore is typically running on port ```9083``` of the Hive Metastore server (```hive.metastore.uris``` property in Hive). It uses a native Thrift API, which is faster compared to WebHCatalog.
    * The WebHCatalog server (formerly called Templeton) is typically running on port ```50111``` on a single server (```templeton.port``` property).
  * HDFS or WebHDFS/HttpFS: All EXASOL nodes need access to the namenode and all datanodes, either via the native HDFS interface (recommended) or via the http REST API (WebHDFS or HttpFS)
    * HDFS (recommended): The namenode service typically runs on port ```8020``` (```fs.defaultFS``` property), the datanode service on port ```50010``` (```dfs.datanode.address``` property).
    * WebHDFS: The namenode service for WebHDFS typically runs on port ```50070``` on each namenode (```dfs.namenode.http-address``` property), and on port ```50075``` (```dfs.datanode.http.address``` property) on each datanode.
    * HttpFS: Alternatively to WebHDFS you can use HttpFS, exposing the same REST API as WebHDFS. It typically runs on port ```14000``` of each namenode. The disadvantage compared to WebHDFS is that all data are streamed through a single service, whereas webHDFS redirects to the datanodes for the data transfer.
  * Kerberos: If your Hadoop uses Kerberos authentication, the UDFs will authenticate using a keytab file. Each EXASOL node needs access to the Kerberos KDC (key distribution center), running on port ```88```. The KDC is configured in the kerberos config file which is used for the authentication, as described in the [Kerberos Authentication](#kerberos-authentication) section.

Steps:
* Build the library for your Hadoop version from source (see section below).
* Upload the jar to a bucket of your choice. This will allow using the jar in the UDFs. See the user manual for how to use BucketFS.
* Run the following statements in EXASOL to create the UDFs:
```
CREATE SCHEMA ETL;

CREATE OR REPLACE JAVA SET SCRIPT IMPORT_HCAT_TABLE(...) EMITS (...) AS
%scriptclass com.exasol.hadoop.scriptclasses.ImportHCatTable;
%jar /buckets/your-bucket-fs/your-bucket/exa-hadoop-etl-udfs-0.0.1-SNAPSHOT-all-dependencies.jar;
/

CREATE OR REPLACE JAVA SET SCRIPT IMPORT_HIVE_TABLE_FILES(...) EMITS (...) AS
%env LD_LIBRARY_PATH=/tmp/;
%scriptclass com.exasol.hadoop.scriptclasses.ImportHiveTableFiles;
%jar /buckets/your-bucket-fs/your-bucket/exa-hadoop-etl-udfs-0.0.1-SNAPSHOT-all-dependencies.jar;
/

CREATE OR REPLACE JAVA SCALAR SCRIPT HCAT_TABLE_FILES(...)
 EMITS (
  hdfs_server_port VARCHAR(200),
  hdfspath VARCHAR(200),
  hdfs_user VARCHAR(100),
  input_format VARCHAR(200),
  serde VARCHAR(200),
  column_info VARCHAR(100000),
  partition_info VARCHAR(10000),
  serde_props VARCHAR(10000),
  import_partition INT,
  auth_type VARCHAR(1000),
  conn_name VARCHAR(1000),
  output_columns VARCHAR(100000),
  debug_address VARCHAR(200))
 AS
%scriptclass com.exasol.hadoop.scriptclasses.HCatTableFiles;
%jar /buckets/your-bucket-fs/your-bucket/exa-hadoop-etl-udfs-0.0.1-SNAPSHOT-all-dependencies.jar;
/
```


## Using the Hadoop ETL UDFs

The following examples assume that you have simple authentication. If your Hadoop requires Kerberos authentication, please refer to the [Kerberos Authentication](#kerberos-authentication) section.

Run the following query to show the contents of the HCatalog table sample_07.
```sql
IMPORT INTO (code VARCHAR(1000), description VARCHAR (1000), total_emp INT, salary INT)
FROM SCRIPT ETL.IMPORT_HCAT_TABLE WITH
 HCAT_DB         = 'default'
 HCAT_TABLE      = 'sample_07'
 HCAT_ADDRESS    = 'thrift://hive-metastore-host:9083'
 HDFS_USER       = 'hdfs';
```

Run the following statement to import into an existing table.
```sql
CREATE TABLE sample_07 (code VARCHAR(1000), description VARCHAR (1000), total_emp INT, salary INT);

IMPORT INTO sample_07
FROM SCRIPT ETL.IMPORT_HCAT_TABLE WITH
 HCAT_DB         = 'default'
 HCAT_TABLE      = 'sample_07'
 HCAT_ADDRESS    = 'thrift://hive-metastore-host:9083'
 HDFS_USER       = 'hdfs';
```
The EMITS specification is not required because the columns are inferred from the target table.

### Mandatory Parameters

Parameter           | Value
------------------- | -----------
**HCAT_DB**         | HCatalog Database Name. E.g. ```'default'```
**HCAT_TABLE**      | HCatalog Table Name. E.g. ```'sample_07'```.
**HCAT_ADDRESS**    | (Web)HCatalog Address. E.g. ```'thrift://hive-metastore-host:9083'``` if you want to use the Hive Metastore (recommended), or ```'webhcat-host:50111'``` if you want to use WebHCatalog. Make sure EXASOL can connect to these services (see prerequisites above).
**HDFS_USER**       | Username for HDFS authentication. E.g. ```'hdfs'```, or ```'hdfs/_HOST@EXAMPLE.COM'``` for Kerberos (see Kerberos Authentication below).

### Optional Parameters

Parameter           | Value
------------------- | -----------
**PARALLELISM**     | Degree of Parallelism, i.e. the maximum number of parallel JVM instances to be started for loading data. ```nproc()```, which is the total number of nodes in the EXASOL cluster, is a good default value.
**PARTITIONS**      | Partition Filter. E.g. ```'part1=2015-01-01/part2=EU'```. This parameter specifies which partitions should be loaded. For example, ```'part1=2015-01-01'``` will only load data with value ```2015-01-01``` for the partition ```part1```. Multiple partitions can be separated by ```/```. You can specify multiple comma-separated filters, e.g. ```'part1=2015-01-01/part2=EU, part1=2015-01-01/part2=UK'```. The default value ```''``` means all partitions should be loaded. Multiple values for a single partition are not supported(e.g. ```'part1=2015-01-01/part1=2015-01-02'```).
**OUTPUT_COLUMNS**  | Specification of the desired columns to output, e.g. ```'col1, col2.field1, col3.field1[0]'```. Supports simple [JsonPath](http://goessner.net/articles/JsonPath/) expressions: 1. dot operator, to access fields in a struct or map data type and 2. subscript operator (brackets) to access elements in an array data type. The JsonPath expressions can be arbitrarily nested.
**HDFS_URL**        | HDFS/WebHDFS/HttpFS URL. E.g. ```'webhdfs://hdfs-namenode:50070'``` (WebHDFS) ```'webhdfs://hdfs-namenode:14000'``` (HttpFS) or ```'hdfs://hdfs-namenode:8020'``` (native HDFS). This parameter overwrites the HDFS URL retrieved from HCatalog. Use this if you want to use WebHDFS instead of the native HDFS interface, or if you need to overwrite it with another HDFS address (e.g. because HCatalog returns a non fully-qualified hostname unreachable from EXASOL). Make sure EXASOL can connect to the specified HDFS service (see prerequisites above).
**AUTH_TYPE**       | The authentication type to be used. Specify ```'kerberos'``` (case insensitive) to use Kerberos. Otherwise, simple authentication will be used.
**AUTH_KERBEROS_CONNECTION**        | The connection name to use with Kerberos authentication.
**DEBUG_ADDRESS**   | The IP address/hostname and port of the UDF debugging service, e.g. ```'myhost:3000'```. Debug output from the UDFs will be sent to this address. See the section on debugging below.




## Building from Source

You have to build the sources depending on your Hive and Hadoop version as follows. The resulting JAR with all dependencies is stored in ```target/exa-hadoop-etl-udfs-0.0.1-SNAPSHOT-all-dependencies.jar```.

#### Vanilla Apache Hadoop and Hive (no distribution)
```
mvn clean -DskipTests package assembly:single -P-cloudera -Dhadoop.version=1.2.1 -Dhive.version=1.2.1
```
This command deactivates the Cloudera Maven profile which is active by default.

#### Cloudera CDH
You can look up the version numbers in the CDH documentation.
```
mvn clean -DskipTests package assembly:single -P cloudera -Dhadoop.version=2.5.0-cdh5.2.0 -Dhive.version=0.13.1-cdh5.2.0
```

#### Hortonworks HDP
You can look up the version numbers in the HDP release notes.
```
mvn clean -DskipTests package assembly:single -P hortonworks -Dhadoop.version=2.7.1.2.3.0.0-2557 -Dhive.version=1.2.1.2.3.0.0-2557
```

#### Other Distributions
You may have to add a Maven repository to pom.xml for your distribution. Then you can compile similarly to examples above for other distributions.



## Kerberos Authentication
Connections to secured Hadoop clusters can be created using Kerberos authentication.

Requirements:
* Kerberos principal for Hadoop (i.e., Hadoop user)
* Kerberos configuration file (e.g., krb5.conf)
* Kerberos keytab which contains keys for the Kerberos principal
* Kerberos principal for the Hadoop NameNode (value of ```dfs.namenode.kerberos.principal``` in hdfs-site.xml)

In order for the UDFs to have access to the necessary Kerberos information, a CONNECTION object must be created in EXASOL. Storing the Kerberos information in CONNECTION objects provides the ability to set the accessibility of the Kerberos authentication data (especially the keytab) for users. The ```TO``` field is left empty, the Kerberos principal is stored in the ```USER``` field, and the Kerberos configuration and keytab are stored in the ```IDENTIFIED BY``` field (base64 format) along with an internal key to identify the CONNECTION as a Kerberos CONNECTION.

In order to simplify the creation of Kerberos CONNECTION objects, the [create_kerberos_conn.py](tools/create_kerberos_conn.py) Python script has been provided. The script requires 4 arguments:
* CONNECTION name
* Kerberos principal
* Kerberos configuration file path
* Kerberos keytab path

Example command:
```
python tools/create_kerberos_conn.py krb_conn krbuser@EXAMPLE.COM /etc/krb5.conf ./krbuser.keytab
```
Output:
```sql
CREATE CONNECTION krb_conn TO '' USER 'krbuser@EXAMPLE.COM' IDENTIFIED BY 'ExaAuthType=Kerberos;enp6Cg==;YWFhCg=='
```
The output is a CREATE CONNECTION statement, which can be executed directly in EXASOL to create the Kerberos CONNECTION object. For more detailed information about the script, use the help option:
```
python tools/create_kerberos_conn.py -h
```

You can then grant access to the CONNECTION to UDFs and users:
```sql
GRANT ACCESS ON CONNECTION krb_conn FOR ETL.HCAT_TABLE_FILES TO exauser;
GRANT ACCESS ON CONNECTION krb_conn FOR ETL.IMPORT_HIVE_TABLE_FILES TO exauser;
```
Or, if you want to grant the user access to the CONNECTION in any UDF (which means that the user can access all the information in the CONNECTION--most importantly the keytab):
```sql
GRANT CONNECTION krb_conn TO exauser;
```

Then, you can access the created CONNECTION from a UDF by passing the CONNECTION name as a UDF parameter as described above. Note: The ```hcat-and-hdfs-user``` UDF parameter must be set the NameNode principal, as described above.

Example:
```sql
IMPORT INTO (code VARCHAR(1000), description VARCHAR (1000), total_emp INT, salary INT)
FROM SCRIPT ETL.IMPORT_HCAT_TABLE WITH
 HCAT_DB         = 'default'
 HCAT_TABLE      = 'sample_07'
 HCAT_ADDRESS    = 'thrift://hive-metastore-host:9083'
 HDFS_USER       = 'hdfs/_HOST@EXAMPLE.COM'
 AUTH_TYPE       = 'kerberos'
 AUTH_KERBEROS_CONNECTION = 'krb_conn';
```



## Debugging
To see debug output relating to Hadoop and the UDFs, you can use the Python script [udf_debug.py](tools/udf_debug.py).

First, start the udf_debug.py script, which will listen at the specified address and port and print all incoming text.
```
python tools/udf_debug.py -s myhost -p 3000
```
Then set the ```DEBUG_ADDRESS``` UDF arguments so that stdout of the UDFs will be forwarded to the specified address.
```sql
IMPORT FROM SCRIPT ETL.IMPORT_HCAT_TABLE WITH
 HCAT_DB         = 'default'
 ...
 DEBUG_ADDRESS   = 'myhost:3000';
```
