# Flink with Red Panda and kowl

## Context

## Demo script

* Start RedPanda, Flink, Flink sql client, and Kowl

```sh
docker compose up -d
```

* Kowl User interface [http://localhost:8080](http://localhost:8080)
* Flink User interface [http://localhost:8083](http://localhost:8083)
* Create the items and item.inventory topics

```sh
docker exec -ti flink-kowl-redpanda-1 bash  -c "rpk topic create items"
docker exec -ti flink-kowl-redpanda-1 bash  -c "rpk topic create item.inventory"
docker exec -ti flink-kowl-redpanda-1 bash  -c "rpk topic list"
```

* Start SQL Client

```sh
docker exec -ti flink-kowl-sql-client-1 bash
# in the shell
./sql-client.sh
```

* Create a table to bind to the items topic in RedPanda

```sql
CREATE TABLE items (
    sku STRING,
    storeName STRING,
    id BIGINT,
    price BIGINT,
    quantity BIGINT,
    type STRING,
    ts TIMESTAMP(3),
    proctime AS PROCTIME(),   -- generates processing-time attribute using computed column
    WATERMARK FOR ts AS ts - INTERVAL '5' SECOND
    ) WITH (
        'connector' = 'kafka', 
        'topic' = 'items',  
        'scan.startup.mode' = 'earliest-offset',  -- reading from the beginning
        'properties.bootstrap.servers' = 'redpanda:29092',  
        'format' = 'json' 
    );
```

* Add target table for item.inventory

```
CREATE TABLE item_inventory (
    sku STRING,
    quantity BIGINT,
    ) WITH (
        'connector' = 'kafka', 
        'topic' = 'item.inventory',  
        'properties.bootstrap.servers' = 'redpanda:29092',  
        'format' = 'json' 
    );
```

* Implement the stateful function: compute the inventory per sku cross stores.