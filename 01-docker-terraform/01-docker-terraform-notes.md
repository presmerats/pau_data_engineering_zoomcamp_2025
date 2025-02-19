Docker 
=====
 
Why Docker
----------

TL;DR: 
* Local Experiments 
* Integration tests (CI/CD)
* Reproducibility
* Running pipelines on the cloud (AWS batch, Kubernetes jobs)
* Spark
* Serverless (AWS Lambda, Google functions)

Allows to have isolated and controlled environments. For example we could have:
* a data pipeline in ubuntu, python3.9 and pandas in a container, 
* then a Postgres database in another isolated container
* then a pgAdmin on another environment

Reproducibility, the docker image is a snapshot of the container, with all instruction to build the environment.
You can take this image that you are running locally and run it in another context like in Google Cloud. The container is identical but is running in different scenarios.

Docker examples
---------------

Test docker (run docker desktop or the like first then)
```bash
docker run hello-world
```
Test interactive console from inside the container, un the following command and type linux command line commands once the bash from inside the container is running. 
```bash
docker run -it ubuntu bash
```

Explanation:
* run means running a container
* options -it means interactive(i) and terminal(t) where you can type
* ubuntu is the docker image to use. This is pulled from the official docker registry
* bash is the command to run when firing up the container

How the image restores the environment
```bash
docker run -it ubuntu bash
$rm -rf / 
```
We removed  everything inside the container but when we run again
```bash
docker run -it ubuntu bash
```
We are back to the desired state. It's in isolation so removign all files didn't affect our computer. It's reproducible so the image is always keeping the desired state.

Pipeline Python Script
-----------------------

Exmaple
```bash
docker run -it python:3.9
```
If we want to use pandas, we need to install it. The entrypoint of this python:3.9 is a python prompt. 

We can modify the entrypoint of the container, so that we end up in bash and install pandas.
```bash
docker run -it --entrypoint=bash python:3.9
$ pip install pandas
$ python
>> import pandas
>> pandas.__version__
```
If we stop the container and rerun it again, the installation of pandas is not there. It's not persistent.

We can specify a Dockerfile with the instruction to run when creating a new image.

Dockerfile
----------

Base image
```dockerfile
FROM python:3.9 
```

Instructions to run when building it
```dockerfile
RUN pip install pandas
```

Modify the entrypoint
```dockerfile
ENTRYPOINT ["bash"]
```


Then build the image
```
docker build -t test:pandas .
```
Explanation:
* build an image from docker file
* image name test
* add a tag like pandas test:pandas
* path where to look for the dockerfile '.'

Now you can test it:
```bash
docker run -it test:pandas
$ python
>>> import pandas
>>> pandas.__version___
>>> exit()
$ exit
```

First Data Pipeline
--------------------

A very simple python module that represents our toy pipeline
```python
import pandas as pd

# some fancy stuff with pandas

print('job finished successfully')
```

Adding the file, that lives in the same dir as the dockerfile, to the image.
```dockerfile
WORKDIR /app
COPY pipeline.py pipeline.py 
```
WORKDIR will just cd inside /app 

COPY <source> <destination> will copy the source file from the computer to the destination inside the docker image

Now we can build the new image (the tag is overrriden)
```bash
$ docker build -t test:pandas .
```
And then run and check
```bash
$ docker run -it test:pandas
/app# python pipeline.py
jobb finished successfully
```

Now we want the container to run autonomously and also send it some parameters

First the pipeline.py will accept parameters or command line arguments
```python
import sys

day = sys.argv[1]

```

Then the image will be configure to run this file when spawn
```dockerfile
ENTRYPOINT [ "python", "pipeline.py" ]
```

We can already pass command line arguments:
```bash
$ docker run -it test:pandas 2025-01-21
```


Postgres
------

### 04 Here we run postgres locally with docker.

We will use the official postgres image. We get config options from the docker-compose. 
```bash
$ docker run -it \
    -e POSTGRES_USER="root" \
    -e POSTGRES_PASSWORD="root" \
    -e POSTGRES_DB="ny_taxi" \
    -v "$(pwd)/ny_taxi_postgres_data":/var/lig/postgresql/data \
    -p 5432:5432 \
    postgres:13
```
We configure the postgres user and db, and then we create a physical volume to persist changes between restarts of the container.  It needs the full path?

We map the folder from the computer to a folder in the continaer using -v <computer paht>:<container path>

We also specify the port, for port forwarding, computer_port:container_port

Check what is running
```bash
docker ps
``` 
Connect to postgres with pgcli (python tool)
```bash
pipx install pgcli # didn't work
brew install pgcli
``` 
Now connect (you need to install psycopg_binary):
```bash
pgcli -h localhost -p 5432 -u root -d ny_taxi
```

### 03 Download and load the dataset into our postgres 

The links to the data in csv format is saved in [https://github.com/DataTalksClub/nyc-tlc-data](nyc-tlc-data)
```bash
wget https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/yellow_tripdata_2021-01.csv.gz
gzip -d yellow_tripdata_2021-01.csv.gz
head yellow_tripdata_2021-01.csv
wc -l yellow_tripdata_2021-01.csv
```

For the column meanings there is this [https://www.nyc.gov/assets/tlc/downloads/pdf/data_dictionary_trip_records_yellow.pdf](pdf file)

For the zones lookup table there is another [https://d37ci6vzurychx.cloudfront.net/misc/taxi_zone_lookup.csv](csv file)

We can use jupyter notebook or visual studio code (9:37). Interesting pandas features:
* pd.io.sql.get_schema(df, name='desired_table_name') will prepare a SQL create table statement will all data types
* pd.to_datetime(df.column_to_transform to datetime)

Create a connection to postgres and then generate a DDL specifici for postgres. We need sqlalchemy and psycopg2, so we better install them in our python env (use poetry or conda)
```bash
$ brew install postgresql
$ conda activate my_env
$ pip install psycopg2-binary sqlalchemy
``` 

Now the pd.io.sql.get_schema uses the connection we will create and outputs a DDL that is working for postgres (for example INT is now BIGINT):
```python
from sqlalchemy import create_engine
engine = create_engine('postgresql://root:root@localhost:5432/ny_taxi')
print(pd.io.sql.get_schema(df, name='yellow_taxi_data',con=engine))
```
The output is the following
```python
CREATE TABLE yellow_taxi_data (
	"VendorID" BIGINT, 
	tpep_pickup_datetime TIMESTAMP WITHOUT TIME ZONE, 
	tpep_dropoff_datetime TIMESTAMP WITHOUT TIME ZONE, 
	passenger_count BIGINT, 
	trip_distance FLOAT(53), 
	"RatecodeID" BIGINT, 
	store_and_fwd_flag TEXT, 
	"PULocationID" BIGINT, 
	"DOLocationID" BIGINT, 
	payment_type BIGINT, 
	fare_amount FLOAT(53), 
	extra FLOAT(53), 
	mta_tax FLOAT(53), 
	tip_amount FLOAT(53), 
	tolls_amount FLOAT(53), 
	improvement_surcharge FLOAT(53), 
	total_amount FLOAT(53), 
	congestion_surcharge FLOAT(53)
)
```


We need to insert the full data, but it is too big and we will use batches, leveragin pandas iterators.
```python
df_iter = pd.read_csv('yellow_tripdata_2001-01.csv',iterator=True, chinksize=100000)
df = next(df_iter)
# transform any fields
df.tpep_pickup_datetime =  pd.to_datetime(df.tpep_pickup_datetime)
df.tpep_dropoff_datetime = pd.to_datetime(df.tpep_dropoff_datetime)
# we create a table without rows first
df.head(n=0).to_sql(name='yellow_taxi_data', con=engine, if_exists='replace)
# then insert
%time df.to_sql(name='yellow_taxi_data', con=engine, if_exists='append')
```

Check with pgcli
```sql
describe yellow_taxi_data;
select count(*) from yellow_taxi_data;
```

Now iterate all the data with while and iterator. It's not very elegant as it will finishi with an exception (StopIteration) but it works.

```python
from time import time 
while True:
    t_start = time()
    df = next(df_iter)
    df.tpep_pickup_datetime =  pd.to_datetime(df.tpep_pickup_datetime)
    df.tpep_dropoff_datetime = pd.to_datetime(df.tpep_dropoff_datetime)
    df.to_sql(name='yellow_taxi_data', con=engine, if_exists='append')
    t_end = time()
    print('inserted another chunk..., took %.3f seconds' % (t_end - t_start))
```

### 04 PgAdmin

We can check the data ingestion
```sql
select max(tpep_pickup_datetime),min(tpep_pickup_datetime), max(total_amount) FROM yellow_taxi_data;
```
Some strange data
* min time 2008-12-31 -> might be wrong
* max amount is 7661.28USD looks like a lot

Anyway it is better to use pgAdmin 
```bash
docker run -it \
    -e PGADMIN_DEFAULT_EMAIL="admin@admin.com" \
    -e PGADMIN_DEFAULT_PASSWORD="root" \
    -p 8080:80 \
    dpage/pgadmin4
```

Then open browser on localhost:8080 and log in using email and password defined in the docker run command.

Then we need to create a connection to the postgres database:
* host: localhost -> this means the container itself!
* port : -> they can't see each other!
* database
* user root
* password root 

We need a network where both containers see each other.
```bash
docker network create pg-network
```
Now rerun the containers with --network=pg-network
```bash
$ docker run -it \
    -e POSTGRES_USER="root" \
    -e POSTGRES_PASSWORD="root" \
    -e POSTGRES_DB="ny_taxi" \
    -v "$(pwd)/ny_taxi_postgres_data":/var/lig/postgresql/data \
    -p 5432:5432 \
    --network=pg-network \
    --name pg-database \
    postgres:13
```

Check the data is still present
```bash 
pgcli 
select max(tpep_pickup_datetime),min(tpep_pickup_datetime), max(total_amount) FROM yellow_taxi_data;
```
In my case it is not, so I run the container for postgres with the network and name options, then rerun the upload-data.ipynb notebook then recheck again.


Now rerun pg-admin
```bash
docker run -it \
    -e PGADMIN_DEFAULT_EMAIL="admin@admin.com" \
    -e PGADMIN_DEFAULT_PASSWORD="root" \
    -p 8080:80 \
    --network=pg-network \
    --name pgadmin dpage/pgadmin4
```



### 05 Putting the ingestion script into docker

Convert the notebook to a python script, using jupyter
```bash
jupyter nbconvert --to=script upload-data.py
```
Then we clean the script and add argparse command line arguments, and a simple way to run wget from the script.

We can check by droping the table and re-running the script.
```sql
drop table yellow_taxi_data;

select count(1) from yellow_taxi_data;
```

Then we can call the ingestion script 
```bash 
URL="https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/yellow_tripdata_2021-01.csv.gz"  && python ingest_data.py \
    --user=root \
    --password=root \
    --host=localhost \
    --port=5432 \
    --db=ny_taxi \
    --table_name=yellow_taxi_data \
    --url=${URL}
```
THe password should be passed as an env variable so that it is not saved in history or in the github commits.

We also adapt the Dockerfile to install dependencies, and also the name of the script to ingest_data.py
```dockerfile
RUN apt-get install wget
RUN pip install pandas sqlalchemy psycopg2
```

And rebuild
```bash 
docker build -t taxi_ingest:v001 .
```

Run and test
```bash
URL="https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/yellow_tripdata_2021-01.csv.gz" && docker run -it --network=pg-network taxi_ingest:v001 \
    --user=root \
    --password=root \
    --host=pg-database \
    --port=5432 \
    --db=ny_taxi \
    --table_name=yellow_taxi_data \
    --url=${URL}
```
The database is rebuild, remember that there is a reset parameter in pd.to_sql() 


Curiosity, python http.server spawns a http fileserver on the folder , so if you have the file on the computer you can point the docker container to this local address to download by wget
```bash 
$ python -m http.server
```

### Docker compose
Docker compose allows to configure and run a set of containers and networks from a single configuration file in yaml

In win, mac docker-compose is part of docker-desktop. In linux you need to install it separately

* We can to set the version
* we need to specify the services (containers)
* the env vars
* volume mapping
* network mapping, all containers inside services become part of the same docker network

Finally spawn the containers (stop the previous containers first)
```bash 
docker-compose up
```

To shutdown
```bash
docker-compose down
```

To run in dettached mode
```bash 
docker-compose up -d
```

### Docker networking

Elements:
* Ubuntu host
    * docker compose network
        * pgdatabase container
        * pgadmin container

docker ps to see all running containers 

docker network ls 
    * uses name <directory name> _<default>

port mapping
    * -p 5432:5432 this is <host machine port>:<container port>
    * -p 8080:80  this is <host machine port>:<container port>
    * to connect pgcli to postgres we will do localhost:5432

If we already have a postgres in localhost running at port 5432, we will map our postgres container with -p 5431:5432 to locahost:5431
    * now we'll run pgcli -h localhost -p 5431

Now we will add the ingestion pipeline in the same docker file. We have a problem when spawning the ingestion container as it cannot see the contents of the ny_taxi_posrtres. The solucion is to use a .dockerignore file
1. use .dockerignore that lists ./ny_taxi_postgres_data  and run the ingestion pipeline container. THAT DIDN'T WORK!
2. create a folder owned by me "./data" , set the volume mounted to there in docker-compose, and add this to the .dockeringore file

Run again the ingestion script
```bash
# first look at the name of the docker compose network
$ docker network ls
# it's 05_docker_compose_default
# since it is in the same network, we use the postgrest container port  not the port mapped in the host.
URL="https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/yellow_tripdata_2021-01.csv.gz" && docker run -it --network=05_docker_compose_default taxi_ingest:v001 \
    --user=root \
    --password=root \
    --host=pgdatabase \
    --port=5432 \
    --db=ny_taxi \
    --table_name=yellow_taxi_data \
    --url=${URL}
```

Then open pgAdmin in localhost:8080 , connection with host pgdatabase, port 5432 and user and password you know.

But for pgcli from the host computer
```bash
pgcli -h localhost -p 5431 -u root -p 
```

We could remove the portmapping to localhost:5431 and the ingestion pipeline container could still see the pgdatabase:5432 from inside the docker network

SQL
====

Import zones csv into the zones table in our running database. 

Join yellow_taxi_data to zones 
This is a cross join? selects all combinations from all the tables in the from clause
The where clause limits what combinations to show -> INNER JOIN
```sql 
SELECT 
    t.lpep_pickup_datetime, 
    t.lpep_dropoff_datetime, 
    t.total_amount,
    zpu."Borough" || '/' || zpu."Zone" AS "pickup_loc",
    zdo."Borough" || '/' || zdo."Zone" AS "dropoff_loc"
from
    green_taxi_data t, 
    zones zpu,
    zones zdo
WHERE 
    t."PULocationID" = zpu."LocationID"
    AND 
    t."DOLocationID" = zdo."LocationID"
```

INNER JOIN with JOIN operator. They are not equivalent as the where is applied after the join? But in the end they are equivalent probably
```sql 
SELECT 
    t.lpep_pickup_datetime, 
    t.lpep_dropoff_datetime, 
    t.total_amount,
    zpu."Borough" || '/' || zpu."Zone" AS "pickup_loc",
    zdo."Borough" || '/' || zdo."Zone" AS "dropoff_loc"
from
    green_taxi_data t, 
    join zones zpu on t."PULocationID" = zpu."LocationID"
    join zones zdo on t."DOLocationID" = zdo."LocationID"
```

Verifications:

Any PULocationID null?
```sql 
SELECT 
    t.lpep_pickup_datetime, 
    t.lpep_dropoff_datetime, 
    t.total_amount,
    t."PULocationID",
    t."DOLocationID"
from
    green_taxi_data t
WHERE 
    "PULocationID" is NULL
```

Any DOLOcationID Null??
```sql 
SELECT 
    t.lpep_pickup_datetime, 
    t.lpep_dropoff_datetime, 
    t.total_amount,
    t."PULocationID",
    t."DOLocationID"
from
    green_taxi_data t
WHERE 
    "DOLocationID" is NULL
```

Any Id that are not present in the trips? USE not in or left join with null values
```sql
select 
    t."PULocationID"
    , z."LocationID"
    , t."DOLocationID"
    , z2."LocationID"
from 
    green_taxi_data t
    left join zones z on z."LocationID" = t."PULocationID"
    left join zones z2 on z2."LocationID" = t."DOLocationID"
WHERE
    z."LocationID" is NULL 
    or
    z2."LocationID" Is NULL
    
```

Video version with subselect I don't like ti
```sql 
SELECT 
    t.lpep_pickup_datetime, 
    t.lpep_dropoff_datetime, 
    t.total_amount,
    t."PULocationID",
    t."DOLocationID"
from
    green_taxi_data t
WHERE 
    "DOLocationID" NOT IN (select "LocationID" from zones)
```

We remove one location to see use cases for left joins
```sql
DELETE FROM zones where "LocationID" = 142
```

With this remove 142 zone, the previous query will show records with DOLocationID = 142
```sql
select 
    t."PULocationID"
    , z."LocationID"
    , t."DOLocationID"
    , z2."LocationID"
from 
    green_taxi_data t
    left join zones z on z."LocationID" = t."PULocationID"
    left join zones z2 on z2."LocationID" = t."DOLocationID"
WHERE
    z."LocationID" is NULL 
    or
    z2."LocationID" Is NULL
```

```sql
SELECT 
    t.lpep_pickup_datetime, 
    t.lpep_dropoff_datetime, 
    t.total_amount,
    zpu."Borough" || '/' || zpu."Zone" AS "pickup_loc",
    zdo."Borough" || '/' || zdo."Zone" AS "dropoff_loc"
from
    green_taxi_data t
    join zones zpu on t."PULocationID" = zpu."LocationID"
    join zones zdo on t."DOLocationID" = zdo."LocationID"
```

Now we use left join and coalsece
```sql
SELECT 
    t.lpep_pickup_datetime, 
    t.lpep_dropoff_datetime, 
    t.total_amount,
    coalesce(zpu."Borough",'unknown') || '/' || coalesce(zpu."Zone",'unknown') AS "pickup_loc",
    zdo."Borough" || '/' || zdo."Zone" AS "dropoff_loc"
from
    green_taxi_data t
    left join zones zpu on t."PULocationID" = zpu."LocationID"
    left join zones zdo on t."DOLocationID" = zdo."LocationID"
```

Other concepts that are seen:
* RIGHT JOIN
* OUTER JOIN set union
* DATE_TRUNC('day', lpep_pickup_datetime)
* cast(tpepe_dropoff_datetime as DATE) 
* GROUP BY
* ORDER BY using ASC or DESC
* Aggreagtions: COUNT(1), MAX(), 
* GROUP BY multiple fields using column numbers




Terraform
=========

Terraform overview
-----------------

IaS
* simplicity in keeping track of infra
* easier collaboration (github review, pull req, deploy)
* reproducibility
* ensure resources are removed
* DOES NOT deploy or update code
* DOES NOT change immutable resources (change virtual machine type, move data stored,)
* DOES NOT Managed resources not defined in your terraform file (things you create outside will not be controlled by terraform)

Components
* locally you install terraform files to connect to a cloud provider
* You need authorization

Provides
* code that allows you to comunicate and manage resources in cloud providers/platforms:
    * AWS, Azure, GCP
    * Kubenetes
    * VSphere
    * Alibaba Cloud
    * Oracle Cloud Infrastructure
    * Active Directory

Key Terraform commads:
* **init** - get me the providers I need. Will get the providers code and install it in your machine
* **plan** - plan what resources are going to be created
* **apply** - actaully build the infra described in the tf files 
* **destroy** - remove all infrastructure defined

Terraform Basics: Simple one file Terraform Deployment
-----------------------------



GCP
====

Services
* compute
* management 
* networking 
* storage & databases 
* Big data 
* id and securtiy
* ML

We will mostly use Storage and Big data

New account -> 300$ free

Lateral Meny

Cloud Storage -> create bucket(cloud storage = S3 AWS)

USE THe search bar

Big Query 
    taxi-rides-ny
        -ntaxy db
            - green_tripdata_2019
            - yellow_tripdate_2019 

        those will be generated during the course


GCP Setup
----------

### GCP account setup & instance

[SOURCE](https://www.youtube.com/watch?v=ae-CV2KfoN0&list=PL3MmuxUbc_hJed7dXYoJw8DoCuVHhGEQb&index=15)

new google account + credit card = 289$ to use in 90days

Go to VM -> enable API

Create SSH keys [GCP Create SSH keys instruccions](https://cloud.google.com/compute/docs/connect/create-ssh-keys):

* Choose authentication (?)
* generate the keys ```bash ssh-keygen -t rsa -f ~/.ssh/de_zoomcamp_gcp_key -C dezoomer ```
* Go to Cloud Engine/MEtadata/Add ssh keys


Create an instance:

* VM instances /Create instance
* Machine configuration
  * name + zone - >europe berlin (green energy)
  * 4cpu (2 cores) 16GB -> 21cents per hour, 150 per month
* OS and storage:
  * set bootdisk to ubuntu(optional)
  * 30GB since we are downloading a lot of files
* NO more changes -> CREATE
* copy external ip
* then ssh into it ```bash ssh -i ~/.ssh/de_zoomcamp_gcp_key deezoomer@XX.XX.XX.XX```


* Add ~/.ssh/config
  * open in vscode
  * configure access to this ssh 
```bash
Host de-zoomcamp
  Hostname XX.XX.XX.XX
  User dezoomer
  IdentityFile ~/.ssh/de_zoomcamp_gcp_key
```



### Install needed software

* htop to check
* download Anaconda
  * Anaconda webpage -> wget command 
  * logout and login or source .bashrc

install docker 
```bash
sudo apt-get update 
sudo apt-get install docker.io
```

vscode to access this machine
* extensions /remote ssh (from Microsoft)
* command+shift+P / Connect to host / -> use the ssh config 

Clone github repo ```bash git clone https://github.com/DataTalksClub/data-engineering-zoomcamp.git```


configure docker:
* [docker without sudo ](https://docs.docker.com/engine/install/linux-postinstall/)
```bash
sudo groupadd docker
sudo usermod -aG docker $USER
````

Now logout and log back or run  ```bash newgrp docker```

Then verify the installation
```bash
docker run hello-world
```
Install [docker-compose](https://github.com/docker/compose?tab=readme-ov-file#linux):
```bash
mkdir bin
cd bin
wget https://github.com/docker/compose/releases/download/v2.33.0/docker-compose-linux-x86_64 -O docker_compose
chmod +x docker_compose
```
Check it's executable
```bash
./docker_compose version
````

Update PATH envvar in .bashrc file wiht ```bash vim .bashrc ``` add at the end:
```bash
export PATH="${HOME}/bin:${PATH}"
```

Then you can test it, first do ```bash source ~/.bashrc```then ```bash docker-compose version```

Then check
```bash 
cd 01-docker-terraform/2_docker_sql/
docker-compose up -d
```

check with 
```bash
docker ps
```

Now install pgcli
```bash
pip install pgcli
pgcli
>\dt
```
And you can see the schema there.

Or install with conda
```bash
pip uninstall pgcli
conda install -c conda-forge pgcli
# not needed for me
# pip install -U mycli 
```

Then check it 
Using `pgcli` to connect to Postgres

```bash
pgcli -h localhost -p 5432 -u root -d ny_taxi
```

### Port forwarding

Setup port forwarding to a local machine now

From the vscode instance connected to the remote machine, 
* open the termina (Ctrl+tilda) 
* then click on ports 
* then add port forwarding 5432 to localhost:5432
* then test locally ```bash pgcli -h localhost -p 5432 -u root -d ny_taxi``` 


REview of port forwarding:

Local port forwarding (my local port is forwarded to the remote machine). This example opens a connection to the gw.example.com jump server, and forwards any connection to port 80 on the local machine to port 80 on intra.example.com:
```bash
ssh -L 80:intra.example.com:80 gw.example.com
```
For the pgadmin it is 
```bash 
ssh -L 8080:localhost:8080 de-zoomcap
```
So it connect to de-zommcamp, and forward a local connection to 8080 to localhost on de-zoomcamp:8080


REmote port forwarding, remote machine can connect to his/her localhost:80 and this will be forwarded to localhost:8080
```bash
ssh -R 8080:localhost:80 public.example.com
```

### Jupyter notebook

Go to  week 1 2_docker_sql folder and run jupyter notebook
```bash
cd 01-docker-terrafomr/2_docker_sql
jupyter notebook
```

Copy the url with the token, then create another port forwarding at port 8888.

Download the data
```bash
wget https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/yellow_tripdata_2021-01.csv.gz
gzip -d yellow_tripdata_2021-01.csv.gz
head yellow_tripdata_2021-01.csv
wc -
```

Install psycopg2
```bash
 pip install psycopg2-binary
 ``` 

Then run the notebook
* create the schema and check with local connection to pgcli
* insert 100 rows and check


### Terraform install

1. Go to [](https://developer.hashicorp.com/terraform/install#linux) and copy link to amd64 executable file for linux
2. Wget this file into our /bin folder
3. sudo apt-get install unzip and unzip the file


We need the credentials terraform or GCP file ? WHere is that?
* follow the instructions here: data_engieering_zoomcamp/01-docker-terraform/1_terraform_gcp/2_gcp_overview.md
* GCP copy project ID
* GCP service account impersonation [link](https://cloud.google.com/docs/authentication/use-service-account-impersonation)


We are going to setup the glcloud api/service-account within the instance, so that it can use terraform to run comands right?

#### Initial Setup

For this course, we'll use a free version (upto EUR 300 credits). 

1. Create an account with your Google email ID 
2. Setup your first [project](https://console.cloud.google.com/) if you haven't already
    * eg. "DTC DE Course", and note down the "Project ID" (we'll use this later when deploying infra with TF)
3. Setup [service account & authentication](https://cloud.google.com/docs/authentication/getting-started) for this project
    * Grant `Viewer` role to begin with. (Don't know how to do it, see the Setup for Access part)
    * Download service-account-keys (.json) for auth. (inside IAM/Service Account you can download them)
4. Download [SDK](https://cloud.google.com/sdk/docs/quickstart) for local setup
5. Set environment variable to point to your downloaded GCP keys:
   ```shell
   export GOOGLE_APPLICATION_CREDENTIALS="<path/to/your/service-account-authkeys>.json"
   
   # Refresh token/session, and verify authentication
   gcloud auth application-default login
   ```
   
#### Setup for Access
 
1. [IAM Roles](https://cloud.google.com/storage/docs/access-control/iam-roles) for Service account:
   * Go to the *IAM* section of *IAM & Admin* https://console.cloud.google.com/iam-admin/iam
   * Click the *Edit principal* icon for your service account.
   * Add these roles in addition to *Viewer* : **Storage Admin** + **Storage Object Admin** + **BigQuery Admin**
   
2. Enable these APIs for your project:
   * https://console.cloud.google.com/apis/library/iam.googleapis.com
   * https://console.cloud.google.com/apis/library/iamcredentials.googleapis.com
   
3. Please ensure `GOOGLE_APPLICATION_CREDENTIALS` env-var is set.
   ```shell
   export GOOGLE_APPLICATION_CREDENTIALS="<path/to/your/service-account-authkeys>.json"
   ```

Obs: the path to your service-account-authkeys was:

* /Users/pau/.gc/hypnotic-runway-451217-r0-0b3a826d7875.json

And has been changed to 

* /Users/pau/.config/gcloud/application_default_credentials.json


 

Now we transfer the json credential files. Put yourself in the folder with the json credentials file and then:
```bash
sftp de-zoomcamp
mkdir .gc
cd .gc
put hypnotic-runway-451217-r0-0b3a826d7875.json
```

stoped at 42:50 minut


Now ssh into the instance and run google cloud authentication for gcloud:
```bash
# first setup the GOOGLE_APPLICATION_CREDENTIALS
echo "export GOOGLE_APPLICATION_CREDENTIALS=/home/dezoomer/.gc/hypnotic-runway-451217-r0-0b3a826d7875.json" >> /home/dezoomer/.bashrc
# Doesn't work anymore
#gcloud auth authenticate-service-account --key-file $GOOGLE_APPLICATION_CREDENTIALS
gcloud auth application-default login
```
It says that it is already authenticated?
```
You are running on a Google Compute Engine virtual machine.
The service credentials associated with this virtual machine
will automatically be used by Application Default
Credentials, so it is not necessary to use this command.

If you decide to proceed anyway, your user credentials may be visible
to others with access to this virtual machine. Are you sure you want
to authenticate with your personal account?
```

I will still authenticate¿ and run terraform


#### Terraform tests 

```bash
cd data-engineering-zoomcamp/01-docker-terraform/1_terraform_gcp/terraform/terraform_basic/
terraform init
terraform plan
```

What I need to do is the following:

* ~~copy the terraform with variables folder to this personal repo?~~
* ~~IN fact copy the rest of the repo into this repo, so that everything is here?~~
* modify the variables accordingly
* push to github ? (so it is a backup)
* git clone into the instance 
* then run terraform from there



Github Codespaces
================




Environments
============


Local
-----



Codespaces
----------



GCP
---



