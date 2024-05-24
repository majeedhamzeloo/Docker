# How to Setup PostgreSQL Replication with Docker-Compose
#### In this setup we will deal with only one replica for simplicity sake. The schematic representation of the setup we aim to achieve is presented below.
![alt text](image.png)

#### The setup for replicated PostgreSQL includes two files: 00_init.sql initialization SQL script and docker-compose.yaml with description of primary and replica docker services. The structure of the catalog is presented below.
```bash
.
├── 00_init.sql
└── docker-compose.yaml
```
#### Initialization script is used to create a replication slot and a user that will use this slot. This file is mounted to the container directory /docker-entrypoint-initdb.d which is a default directory used by postgres:14 image for database initialization scripts. Here is the content of 00_init.sql file:
```bash
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'replicator_password';
SELECT pg_create_physical_replication_slot('replication_slot');
```
#### Two docker-compose services are described in docker-compose.yaml file. The first service is a primary instance of PostgreSQL and the other one is a replica. The detailed explanation of the file content is presented in the following section.
```bash
version: '3.8'
x-postgres-common:
  &postgres-common
  image: postgres:14-alpine
  user: postgres
  restart: always
  healthcheck:
    test: 'pg_isready -U user --dbname=postgres'
    interval: 10s
    timeout: 5s
    retries: 5

services:
  postgres_primary:
    <<: *postgres-common
    ports:
      - 5432:5432
    environment:
      POSTGRES_USER: user
      POSTGRES_DB: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_HOST_AUTH_METHOD: "scram-sha-256\nhost replication all 0.0.0.0/0 md5"
      POSTGRES_INITDB_ARGS: "--auth-host=scram-sha-256"
    command: |
      postgres 
      -c wal_level=replica 
      -c hot_standby=on 
      -c max_wal_senders=10 
      -c max_replication_slots=10 
      -c hot_standby_feedback=on
    volumes:
      - ./00_init.sql:/docker-entrypoint-initdb.d/00_init.sql

  postgres_replica:
    <<: *postgres-common
    ports:
      - 5433:5432
    environment:
      PGUSER: replicator
      PGPASSWORD: replicator_password
    command: |
      bash -c "
      until pg_basebackup --pgdata=/var/lib/postgresql/data -R --slot=replication_slot --host=postgres_primary --port=5432
      do
      echo 'Waiting for primary to connect...'
      sleep 1s
      done
      echo 'Backup done, starting replica...'
      chmod 0700 /var/lib/postgresql/data
      postgres
      "
    depends_on:
      - postgres_primary
```
#### To run primary and replica instances in background you can use following command:
```bash
root@srv01:/source/postgresql# docker-compose up -d postgres_primary postgres_replica
Creating network "postgresql_default" with the default driver
Pulling postgres_primary (postgres:14-alpine)...
14-alpine: Pulling from library/postgres
d25f557d7f31: Pull complete
57103dbfb65d: Pull complete
c0ee2744e27d: Pull complete
b3062d5befd9: Pull complete
97a75adef004: Pull complete
3f734eaee16f: Pull complete
4c45bbf7601b: Pull complete
83adffbd3816: Pull complete
39d26101c9ba: Pull complete
Digest: sha256:c393148bf60528a7591b60af14499ea27bc165e8cb1cffe63bbe1ca1e009dfb1
Status: Downloaded newer image for postgres:14-alpine
Creating postgresql_postgres_primary_1 ... done
Creating postgresql_postgres_replica_1 ... done
root@srv01:/source/postgresql# docker ps
CONTAINER ID   IMAGE                COMMAND                  CREATED          STATUS                             PORTS                                       NAMES
82b6946bbf3e   postgres:14-alpine   "docker-entrypoint.s…"   28 seconds ago   Up 27 seconds (health: starting)   0.0.0.0:5433->5432/tcp, :::5433->5432/tcp   postgresql_postgres_replica_1
bf1b6d13c00e   postgres:14-alpine   "docker-entrypoint.s…"   29 seconds ago   Up 25 seconds (healthy)            0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   postgresql_postgres_primary_1
```
## Testing
#### PostgreSQL has a lot of useful system views that are placed in pg_catalog schema and that are handy for diagnostic/debugging purposes. There are at least couple of views that suits for verifying replication status: pg_replication_slots and pg_stat_replication. Here is an example of command and its output which uses the first mentioned view:
```bash
root@srv01:/source/postgresql# docker exec -it postgresql_postgres_primary_1 bash
736b89959415:/$ psql postgres://user:password@localhost:5432/postgres -xc \
'select * from pg_replication_slots;'
-[ RECORD 1 ]-------+-----------------
slot_name           | replication_slot
plugin              | 
slot_type           | physical
datoid              | 
database            | 
temporary           | f
active              | t
active_pid          | 52
xmin                | 
catalog_xmin        | 
restart_lsn         | 0/3000060
confirmed_flush_lsn | 
wal_status          | reserved
safe_wal_size       | 
two_phase           | f
```
#### Create sample table on primary.
```bash
root@srv01:~# docker exec -it postgresql_postgres_primary_1 bash
                                        ^
736b89959415:/$ psql postgres://user:password@localhost:5432/postgres -xc 'create table test1(id int, name varchar(100));'
CREATE TABLE
                                                ^
736b89959415:/$ psql postgres://user:password@localhost:5432/postgres -xc 'insert into test1 (id) values (1);'
INSERT 0 1
736b89959415:/$ 
exit
```
#### Check table data on secondary.
```bash
root@srv01:~# docker exec -it postgresql_postgres_replica_1 bash
edcfdee91d6e:/$ psql postgres://user:password@localhost:5432/postgres -xc 'select * from test1;'
-[ RECORD 1 ]
id   | 1
name | 
```
## Detailed explanation of docker-compose.yaml
#### At the beginning of docker-compose.yaml there is a section with reused part of service declarations. It uses standard YAML anchor syntax to define a common part.
```bash
x-postgres-common:
  &postgres-common
  image: postgres:14-alpine
  user: postgres
  restart: always
  healthcheck:
    test: 'pg_isready -U user --dbname=postgres'
    interval: 10s
    timeout: 5s
    retries: 5
...
```
#### In this section user is set to fulfill the postgres command requirement. It is required for the replica mostly because instead of default entry-point a custom bash script is executed calling postgres command. To check database status inside container a utility pg_isready is used. Primary instance of PostgreSQL is defined within postgres_primary service section. It declares exposed port (5432, default for PostgreSQL), environment variables, command with extra arguments required for replication and finally mounted docker volume with initialization script.
```bash
...
services:
  postgres_primary:
    <<: *postgres-common
    ports:
      - 5432:5432
    environment:
      POSTGRES_USER: user
      POSTGRES_DB: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_HOST_AUTH_METHOD: "scram-sha-256\nhost replication all 0.0.0.0/0 md5"
      POSTGRES_INITDB_ARGS: "--auth-host=scram-sha-256"
    command: |
      postgres 
      -c wal_level=replica 
      -c hot_standby=on 
      -c max_wal_senders=10 
      -c max_replication_slots=10 
      -c hot_standby_feedback=on
    volumes:
      - ./00_init.sql:/docker-entrypoint-initdb.d/00_init.sql
...
```
#### The subtle thing here is the way how POSTGRES_HOST_AUTH_METHOD is defined. As per docker-entrypoint.sh, the content of this variable is written to pg_hba.conf. This file is used by PostgreSQL to configure access restrictions. It is required to ease default restrictions and allow replica connect to primary instance.
#### Although postgres_replica service uses the same docker image, it has slightly different configuration. First of all, another host port is mapped because the default one is already used by primary
#### Secondly, pair of another environment variables is used. We use POSTGRES_USER / POSTGRES_PASSWORD variables to set superuser credentials for the primary instance and PGUSER/ PGPASSWORD to provide user that is used by pg_basebackup utility to connect to the primary instance.
```bash
...
postgres_replica:
    <<: *postgres-common
    ports:
      - 5433:5432
    environment:
      PGUSER: replicator
      PGPASSWORD: replicator_password
    command: |
      bash -c "
      until pg_basebackup --pgdata=/var/lib/postgresql/data -R --slot=replication_slot --host=postgres_primary --port=5432
      do
      echo 'Waiting for primary to connect...'
      sleep 1s
      done
      echo 'Backup done, starting replica...'
      chmod 0700 /var/lib/postgresql/data
      postgres
      "
    depends_on:
      - postgres_primary
```
#### In command of postgres_replica section the bash script is executed. It periodically polls primary instance with pg_basebackupuntil it finally succeeds. As you can remember, PostgreSQL backup and replication are coupled. To get replication running it is required to prepare the replica’s initial state. It is done by pg_basebackup .
#### Arguments -R and --slot=replication_slot are very important. Argument -R makes it easier to startup replica in constantly recovering mode (what replication essentially is) with single postgres command. Provided this argument, pg_basebackup creates special files standby.signal and postgresql.auto.conf in $PGDATA directory. The first file is a simple switch that makes the server to start in standby mode and enter recovery. The second file postgresql.auto.conf contains connection attributes which instruct replica how to connect to the primary instance. 
#### Other important argument is --slot=replication_slot. It specifies that pg_basebackup utility must use the same replication slot that was created in 00_init.sql script.
#### After pg_basebackup is done the only thing that remains is to configure permissions for $PGDATA directory copied by utility and startup the PostgreSQL replica server. That is what two last lines of Bash script do.

#### To stop all services and remove volumes use this one:
```bash
root@srv01:/source/postgresql# docker-compose down -v
```

##### https://medium.com/@eremeykin/how-to-setup-single-primary-postgresql-replication-with-docker-compose-98c48f233bbf