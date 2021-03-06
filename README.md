
## Table of content
- [Introduction](#introduction)
- [Folder Structure](#folder-structure)
- [Steps to run the code](#steps-to-run-the-code)
    - [Requirements](#requirements)
- [Framework Functionality](#framework-functionality)
  - [What has been achieved from 'points to achieve'](#What-has-been-achieved-from-points-to-achieve)
  - [Workarounds for 'points to achieve'](#Workarounds-for-points-to-achieve)
- [Postgres Table Details with Schema](#postgres-tables-details)
- [Future work-What would you improve if given more days](#future-work)
- [Version of tools used](#Version-of-tools-used)

## Introduction
The intention of this project is to build a non-blocking parallel processing ETL pipeline that would ingest
data from a .csv file, process the data and load to a database table, following which the data from the products
table needs to be aggregated and further inserted into another table. I have used dockerised **Spark**, with scripts written in 
pyspark, as my ETL tool and have used Postgres as the target database.  

<img src=./images/image8.png width="700" height="400">

Following were the points intended to achieve
- Code should follow the concept of OOPS
- Support for regular non-blocking parallel ingestion of the given file into a table. Consider thinking about the scale 
  of what should happen if the file is to be processed in 2 minutes.
- Support for updating existing products in the table based on `sku` as the primary key.
- All product details are to be ingested into a single table
- An aggregated table on above rows with `name` and `no. of products` as the columns

Assumptions made during the development of the ETL framework are as follows :
- No column was provided with input data in products.csv that would allow one to identify the latest update record for a
  particular `sku` value, thus a `request_id` column has been derived in spark, which is a concatenation of timestamp 
  and row number. The assumption that is made is that every data that is input would natively have its own timestamp based 
  on which you can filter out only the latest values.
- One of the other assumptions or inferences that has been made is that it's been expected that update should happen on
target table treating `sku` like a primary key i.e only one record of a `sku` value can exist, but this does not 
  necessarily imply that column `sku` should be made the primary key in the table
  
## Folder Structure
A typical top-level directory layout
```
root/
 |-- configs/
 |   |-- PRODUCTS.yaml
 |-- dependencies/
 |   |-- data_check.py
 |   |-- env_config_parser.py
 |   |-- extract.py
 |   |-- load.py
 |   |-- log.py
 |   |-- postgresql.py
 |   |-- table_config_parser.py
 |   |-- transforms.py
 |   |-- utils.py
 |-- jobs/
 |   |-- etl_job.py
 |-- source_data/
 |   |-- products.csv
 |   code.zip
 |   new.bat
 |   Dockerfile
 |   Dockerfile2
 |   requirements.txt
 |   servers.json
 
```

The main Python module containing the ETL job (which will be sent to the Spark cluster), is `jobs/etl_job.py`. Any external configuration parameters required by etl_job.py are stored in yaml format in `configs/PRODUCTS.yaml` . Additional modules that support this job are kept in the dependencies folder. In the project's root I have included new.bat, which is a batch script to build the docker images. Requirements.txt file contains all the additional modules thats needed by pyspark like `psycopg2` and `PyYAML`


## Steps to run the code
There are three containers in total that host the whole framework on docker. Each for spark, postgresql and pgadmin4. The spark configuration is run as with one master and 3 workers all running on bitnami/spark images to simulate a cluster
### Requirements 
- Docker
- Docker Compose
- Git

1. Clone the code base from this github repo using command

  ```
  git clone https://github.com/pradeeppaikateel/postman-assignment.git 
  ```
2. Change directory to folder `postman-assignment`
```
cd postman-assignment
```
**NOTE: All commands in cmd/terminal are to be run from above directory**

As this framework is built in a windows machine, I have created a batch file that would execute all initial commands.

3. Execute the following command to run the batch file in windows cmd
```
new.bat
```
the batch file contains the following commands that can be run individually **if needed**
```
docker build -t pradeep/spark -f Dockerfile .
docker build -t pradeep/pgadmin4 -f Dockerfile2 .
docker network inspect develop >$null 2>&1 || docker network create --subnet=172.18.0.0/16 develop
docker start postgres > $null 2>&1 || docker run --name postgres --restart unless-stopped -e POSTGRES_PASSWORD=postgres --net=develop --ip 172.18.0.22 -e PGDATA=/var/lib/postgresql/data/pgdata -v /opt/postgres:/var/lib/postgresql/data -p 5432:5432 -d postgres:11
docker-compose up -d
docker run -p 5050:80 --volume=pgadmin4:/var/lib/pgadmin -e PGADMIN_DEFAULT_EMAIL=postman@sample.com -e PGADMIN_SERVER_JSON_FILE=/pgadmin4/servers.json -e PGADMIN_DEFAULT_PASSWORD=postman --name="pgadmin4" --hostname="pgadmin4" --network="develop" --detach pradeep/pgadmin4
```
4. Once the above commands are successful , the command given below can be used to submit the pyspark script to trigger the ETL job from local windows machine cmd
```
docker exec spark-master spark-submit --master spark://spark:7077 --py-files /opt/bitnami/spark/postman-assignment/code.zip /opt/bitnami/spark/postman-assignment/jobs/etl_job.py --source=PRODUCTS --env=dev --job_run_date=2021-08-07T02:00:00+00:00
```



## Framework Functionality

### What has been achieved from points to achieve

This spark framework offers the following functionality which covers all the 'points to achieve' as following:
- The framework has been created following OOPS concept and every process is set up as an object like Extract, Transform, Load etc, where each object has its own methods.
- the data is repartitioned to a number of 100, which would ensure that non-blocking parallel procession ensures even as the cluster resources scale up.
- Loading of data into the target table happens using the Upsert functionality treating `sku` as the conflict key, wherein if a record with said `sku` does not exist then it is inserted into target table, and if it does then the other columns are updated.
- All product details are ingested to a single table called as products in database postman.
- An aggregate ETL process is also run wherein the data is taken from the products table , aggregated and inserted into products_agg table of database postman
- Truncation of data does not happen in any of the loading process
- A logger module is implemented which keeps a log of the ETL process (details given later)
- Error handling has been implmented to raise meaningful errors
- A data sanity check is done at the beginning after the extract phase, where a check is done on the count of records (count of records should be greater than 0), and the rows are checked for null values in any of the source columns. If null values are found , it is filtered out and loaded into a csv file to be inspected by developer
- The target table(products) records are also inserted with a `record checksum` ,`update timestamp`, `p id` and `request id` so that the data available in products table can be used by other processes with ease

On execution of the pyspark job, the products and products_agg table will have following count of records:
```
count of records in products: 466693
count of records in products_agg: 212645
```
sample 10 rows from products table:

<img src=./images/image5.png width="900" height="400">

sample 10 rows from products_agg table:

<img src=./images/image6.png width="700" height="400">

Please navigate to [postgres details](#postgres-tables-details) section to connect to database and query/check data


Once the spark job is executed the logs can be seen with the following command
```
docker exec spark-master more /opt/bitnami/spark/logs/spark.txt
```

If error records are found then they are written to the following location
```
/opt/bitnami/spark/postman-assignment/error_records/error_records.csv
```

### Workarounds for points to achieve
Everything has been achieved from the 'points to achieve', there were instructions to include support for updating the products table based on `sku` as the primary key, `sku` has not been made the primary key as its always better to have a numeric as a primary key for ease of querying records and faster performance when dealing with upcoming huge future data. The workaround for this was to generate a surrogate key using zipwithuniqueid() function of rdd to generate surrogate values that do not have a chance of a collision as the data scales.


## Postgres Tables Details

Two tables are created/used , the `products` table and `products_agg` table

products table:
```
CREATE TABLE IF NOT EXISTS postman.public.products( 
p_id bigint primary key,
sku varchar(70) unique not null,
name varchar(70) not null,
description varchar(300) not null,
request_id varchar(40),
record_checksum varchar(45) not null,
updt_tmstmp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
```
drop script
```
DROP TABLE postman.public.products;
```
products_agg table:
```
CREATE TABLE IF NOT EXISTS postman.public.products_agg( 
name varchar(70) primary key,
"no. of products" integer not null
) 
```
drop script
```
DROP TABLE postman.public.products_agg;
```
Details on how to connect to the Postgres server using pgadmin is given below:

1. Go to a browser and connect to :
```
http://localhost:5050/
```
2. Enter email and password as :
```
emailid: postman@sample.com
password: postman
```
<img src=./images/image2.png width="600" height="400">
3. Once logged in, connect to new server with following details:

```
"name": postman
"user": postgres,
"password": postgres,
"host": 172.18.0.22,
"port": 5432,
```

<img src=./images/image1.png width="600" height="400">
<img src=./images/image3.png width="400" height="400">
<img src=./images/image4.png width="400" height="400">

4. Once connected, navigate to the postman database as shown , right click and choose "query tool"

<img src=./images/image7.png width="600" height="400">

5. The following queries can be used to query processed data:

products table
```
SELECT p_id, sku, name, description, request_id, record_checksum, updt_tmstmp FROM postman.public.products;
```
products_agg table
```
SELECT name, "no. of products" FROM postman.public.products_agg;
```

## Future work
If more time was available, the following could have been implemented
- A more thorough data sanity check class and methods for source data
- More detailed error handling framework
- Would have worked on making the framework more source data agnostic, where the framework would be independent of source object and would work for any and all source objects with just change in the parameters of the config file.

## Version of tools used
- Python 3.6.14
- Spark 3.1.2
- Postgres 11
- Pgadmin4
