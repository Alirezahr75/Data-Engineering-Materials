# Debezium

Debezium is a simple and open-source tool for listening to Postgres changes and propagating them on Kafka topics.

## Installation

If you already have Kafka and Postgres available, use docker-compose.yml to install Postgres-Connector. You only need to set the Kafka BOOTSTRAP_SERVERS address in the docker-compose file.

If you don't, just rename docker-compose(Postgres + Kafka + zookeper).yml to docker-compose.yml and use the following commands. 

```bash
export HOST_IP=$(ifconfig | grep -E "([0-9]{1,3}\.){3}[0-9]{1,3}" | grep -v 127.0.0.1 | awk '{print $2}' | cut -f2 -d: |head -n1)
docker-compose up
```

## Debezium and Postgres Configurations
Use below simple JSON file to config debezium (you need to make some changes based on your configuration).

```JSON
{
    "name": "debezium-develop",
    "config": {
      "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
      "plugin.name": "pgoutput",
      "publication.name": "dbz_publication",
      "database.server.name": "postgres",
      "database.server.id": 1234,
      "database.hostname": "mypostgres-postgresql.postgres",
      "database.port": 1234,
      "database.user": "postgres",
      "database.password": "postgres",
      "database.dbname": "my_database",
      "table.include.list": "my_schema.table1, my_schema.table2, my_schema.table3",
      "heartbeat.interval.ms": 300000,
      "heartbeat.topics.prefix": "debezium_heartbeat",
      "heartbeat.action.query": "UPDATE my_schema.debezium_heartbeat SET updated_at=now();",
      "tasks.max": 1,
      "max.batch.size": 4096,
      "max.queue.size": 16384,
      "poll.interval.ms": 10000
    }
}
```

Copy these into a file and save the file under debezium-config.json.

The following will start a connector that reads the tables specified inside "table.include.list" out of the source Postgres database:

```bash
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:9090/connectors/ -d @debezium-config.json
```

If everything was done correctly, This should return a response with the same JSON object you sent.

Finally, by using the following post request, you can find your existing configs:

```bash
curl -H "Accept:application/json" localhost:9090/connectors/
```

## Some Issue
You need to set Postgres wal_level to logical.

Be aware that in this debezium release, you cant change configs after Applying it.

Debezium creates a separate topic for each table in Postgres. So, if you want debezium to do it automatically, set "auto.create.topics.enable" to true in your Kafka config file.


## Postgres Management and Possible Errors
With every change in the database, a wal file is created to keep the current state. If these files are not managed, the disk space of Postgres will be full, and its execution will stop.

The Debezium tool stores its last used wal in a Postgres view named "pg_replication_slots." To find these values, use the following command:
```sql
 SELECT * FROM pg_replication_slots;
```
The above command returns two values, "restart_lsn" and "confirmed_flush_lsn." The value of "confirmed_flush_lsn" indicates the last wal used by the debezium. Also, the "restart_lsn" value specifies the oldest wal needs to be kept. Therefore, to remove wals, the value of restart_lsn should be considered.

Also, you can find stored object file name in your system with the following query:
```sql
SELECT slot_name,
       lpad((pg_control_checkpoint()).timeline_id::text, 8, '0') ||
       lpad(split_part(restart_lsn::text, '/', 1), 8, '0') ||
       lpad(substr(split_part(restart_lsn::text, '/', 2), 1, 2), 8, '0')
       AS wal_file
FROM pg_replication_slots;
```

If a wal file is deleted whose value is newer than the value of "restart_lsn" debezium will encounter the following error:
```bash
receive data from WAL stream:FATAL:  requested WAL segment 0000000800002A0000000000 has already been removed
```

To overcome the above error, The information about debezium in the pg_replication_slots view in Postgres must be deleted and recreated using the follwing commands:
```sql
SELECT pg_drop_replication_slot('debezium');
SELECT * FROM pg_create_logical_replication_slot('debezium', 'pgoutput');
```

In the end, to avoid taking up too much disk space by wal files, "heartbeat.interval.ms" and "heartbeat.action.query" values should be set in the debezium config file so that a dummy query can be performed periodically at specific intervals. This query will update the "restart_lsn" value, and confirmed_flush_lsn values and old wals will be deleted by Postgres.

## Contributing
Pull requests are welcome. 

For major changes, please open an issue first to discuss what you would like to change.
