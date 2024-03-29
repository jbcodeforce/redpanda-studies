# Some use cases / demonstrations

## Pinot integration with RedPanda for flight records

Inject flights data to RedPanda then to Pinot. [See this article](https://redpanda.com/blog/streaming-data-apache-pinot-kafka-connect-redpanda/).

* Start redpanda and Pinot

```sh
# under rpk-pinot-scenario
docker compose up -d
```

The pinot-ex folder is mounted inside the docker container of redpanda and pinot controller.

* [flights-schema.json](https://github.com/jbcodeforce/redpanda-studies/blob/master/rpk-pinot-scenario/pinot-ex/flights-schema.json) to define the flights table in Pinot

* Get the flights data

```sh
 curl -X GET https://raw.githubusercontent.com/systemcraftsman/redpanda-apache-pinot-demo/main/resources/data/flights-data.json > flights-data.json
```

* Create topic in redpanda:

```sh
docker exec -ti redpanda-1 bash
rpk topic create flights
```

* Create table in Pino

```sh
docker exec -ti pinot-controller bash
./bin/pinot-admin.sh AddTable     -schemaFile /tmp/panda_airlines/flights-schema.json     -tableConfigFile /tmp/panda_airlines/flights-table-realtime.json     -exec
```

* Produce message to the `flights` topic

```sh
rpk topic produce flights < /tmp/panda_airlines/flights-data.json
```

* Connect to the pinot web console 

```sh
chrome localhost:9000
```

* Execute the following query in the `Query Console` to verify the connection to Kafka has worked and the table in Pinot is loaded with records from `flights` topic:

```sql
select * from flights limit 10
```

* Then execute the following SQL to address the analyst's request: *Find the number of flights that occurred in January 2014 that have air time of more than 300 minutes and that are from any airport in the state of California to JFK airport.*

```sql
select count(*) from flights
where Dest='JFK'
  and AirTime > 300
  and OriginStateName='California'
  and Month=1
  and Year=2014
```

