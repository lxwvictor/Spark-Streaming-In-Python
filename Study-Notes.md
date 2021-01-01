# Setting kafka config
1. Uncomment below line from server.properties to always use port 9092
    ```
    # listeners=PLAINTEXT://:9092
    ```
1. Set log.dirs from servers.properties
    ```
    log.dirs=D:\\app\\kafka-logs
    ```
1. Set dataDir from zookeeper.properties
    ```
    dataDir=D:\\app\\zookeeper`
    ```

# Start and stop zookeeper, kafka
Reference: https://kafka.apache.org/documentation/#quickstart

## Start zookeeper, then kafka server, from separate consoles and let them running
    zookeeper-server-start.bat $KAFKA_HOME/config/zookeeper.properties  # "INFO binding to port 0.0.0.0/0.0.0.0:2181" from the console means successful start
    kafka-server-start.bat $KAFKA_HOME/config/server.properties # "[KafkaServer id=0] started (kafka.server.KafkaServer)" from the console means successful start

## Stop kafka server and zookeeper
    kafka-server-stop.bat
    zookeeper-server-stop.bat

## Create topic
    kafka-topics.bat --create --topic quickstart-events --bootstrap-server localhost:9092   # "Created topic quickstart-events" from the console means successful creations of topic

## Start consumer and leave it running
    kafka-console-consumer.bat --topic quickstart-events --from-beginning --bootstrap-server localhost:9092

## Start producer and key in some message, the consumer will show the sent message
    kafka-console-producer.bat --topic quickstart-events --bootstrap-server localhost:9092
    hello
    hi

# Run streaming
## Start ncat listener
    ncat -lk 9999

## Run the StreamingWC.py from another console
    cnda tfspark    # activate python environment
    python -m StreamingWC

## Send message from the ncat console and the python console should return work count

## `count()` as from readStream
It's not an action any more. It's a transformation. The object type is a dataframe.

# Streaming Application Model
1. Read Stream
2. Process Stream
3. Write Stream

# Critical Streaming Concepts
1. Sources
2. Sinks
3. Schema
4. Streaming Query
5. Triggers
6. Output Modes
7. Checkpoint
8. Fault Tolerance
9. Exactly Once Processing

# There are 4 micro batch trigger configurations, as of Spark 3.0.0
1. Unspecified - Default, new micro-batch triggered as soon as the current micro-batch finishes, subject to the wait for the new data.
2. Time interval - The second micro batch will wait for the specified time interval if the first micro batch finish in time. If the first batch overrun, it spend more time than the time interval, then the second micro batch will run immediately after the first one finished.
3. One time - Run one time and background progress stops. Good when you want to spin up a Spark cluster in the cloud. It could be better than batch job as you can restart from the same point where you stopped and combining intermediate results.
4. Continuous - To achieve millisecond latency, still experimental

# Spark Streaming Sources, Sinks and Output modes
## Sources
1. Socket source - Good for learning and experimenting, not designed for production.
2. Rate source - Designed for testing and benchmarking Spark cluster. It's a dummy data source which generates a configurable number of key/value pair per second.
3. File source - Most commonly used.
4. Kafka source - Most commonly used.

## Output Modes
1. Append - Insert only. Not supposed for streaming aggregations.
2. Update - Upsert. Updated and new values only.
3. Complete - Overwrite. Full result.

# Fault Tolerance and Restarts
## Checkpoint
1. Read Position
2. State Information

# Streaming from Kafka Source
1. Start the scripts in `03-KafkaStreamDemo/kafka-scripts` in sequence. Copy a few invoice samples from `03-KafkaStreamDemo/datagen/samples.json` and paste to the producer console.
2. Run `python -m KafkaStreamDemo`, the invoice will be processed and saved to `03-KafkaStreamDemo/output`. (Do note the extra `.config("spark.jars.packages", "org.apache.spark:spark-sql-kafka-0-10_2.12:3.0.1") \` to use extra jars)
3. Copy another invoice data and the streaming engine will pick it up in 1 minute, result will be reflected in `03-KafkaStreamDemo/output`.

# Work with Kafka Sinks
One commonly used trick to deal with data transformation and debug is to use `spark.read()` instead of `spark.readStream()` so the dataframe can be easily shown.

Kafka accepts key-value pair only, so `notification_df` needs to be transformed to `kafka_target_df`. One possible way to implement: `df.selectExpr("InvoiceNumber as key", """to_json(named_struct('CustomerCardNo', CustomerCardNo))""")`.

1. Start the 01 to 05 scripts in `04-KafkaSinkDemo/kafka-scripts` in sequence. Copy a few invoice samples from `04-KafkaSinkDemo/data/samples.json` and paste to the producer console.
2. Run `python -m KafkaSinkDemo`. (Do note the extra `.config("spark.jars.packages", "org.apache.spark:spark-sql-kafka-0-10_2.12:3.0.1") \` to use extra jars)
3. Run `04-KafkaSinkDemo/kafka-scripts/06-start-consumer.cmd`, the notification received from the consumer console.
4. Copy few more invoice samples and paste to producer console, the consumer console will update accordingly.

# Multi-query Streams Application
There are 2 queries, `notification_writer_query` and `invoice_writer_query`. Two important points to note:
1. Different queries must use different checkpoints, applies for same application and different applications. `option("checkpointLocation", "chk-point-dir/notify")` and `option("checkpointLocation", "chk-point-dir/flatten")` are used respectively.
2. Waiting for one query will halt the other one. `spark.streams.awaitAnyTermination()` used here.

The sequence or running demo is the same as previous section. Now the result will be in both consumer console and the `output` directory.

In this section the `to_json()` method is using `struct(*)` which takes a list of selected columns. While the `named_struct()` from previous demo allows you to renamed the selected columns.

# Create Kafka avro Sinks
It's almost the same as kafka sinks, just need extra jar dependencies `.config("spark.jars.packages", "org.apache.spark:spark-sql-kafka-0-10_2.12:3.0.1,org.apache.spark:spark-avro_2.12:3.0.1") \` when creating the SparkSession object.

The sequence of running the demo is also the same as previous sections. Just not avro is binary format, so the output from consumer console will look odd.

# Working with Kafka avro Source
Almost the same with previous sections, except avro has it's own schema syntax.

# Stateless VS Stateful transformations
1. Stateless transformation; works on current micro batch, `Complete` output mode not supported.
   - select()
   - filter()
   - map()
   - flatMap()
   - explode()
2. Stateful transformation; reply the history state, works on all consumed data; excessive state causes out of memory.
   - grouping
   - aggregations
   - windowing
   - join
3. Aggregations
   - Continuous aggregation -> unmanaged stateful operation to clean the state
   - Time-bound aggregation -> managed stateful operation to clean the state

# Event time and Windowing
## Windowing Aggregates
- Tumbling Time Window
- Sliding Time Window