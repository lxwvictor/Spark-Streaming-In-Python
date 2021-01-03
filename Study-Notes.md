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
- Tumbling Time Window: non-overlapping window
- Sliding Time Window: (can be) overlapping window

# Tumbling Windows aggregate
Spark stream does not support aggregates cross windows. In order to calculate the buy sell total over all transactions. Streaming can be used to process heavy lifting tasks. Create separate application to handle the analytics.

In the particular `TumblingWindowDemo.py` we can change `kafka_df = spark.readStream()` to `kafka_df = spark.read()`, uncomment the part of `final_output_df` and comment the streaming `window_query`. The code will change to batch processing.

# Watermarking your windows
State store is maintained in executor memory, so a late arriving message can also be aggregated to the correct window frame. This leads to the need of windows expiration, if not, we will run into OOM. Such a mechanism is named watermark.

Two key business questions of watermark
- What is the maximum possible delay?
- When later records are not relevant?

Code example of using watermark. Key points:
1. The watermark is before `groupBy`
2. The event column of watermark is the same as `groupBy`
```
window_agg_df = trade_df \
    .withWatermark("CreatedTime", "30 minute") \
    .groupBy(window(col("CreatedTime"), "15 minute")) \
```

Max (event time) - watermark = watermark boundary

Analysis of the particular `WatermarkDemo.py` script.
- `{"CreatedTime": "2019-02-05 10:48:00", "Type": "SELL", "Amount": 600, "BrokerCode": "ABX"}` sets the watermark boundary to `10:18:00`, so Spark expires the time window of `10:00:00` to `10:15:00` and cleans the state store.
- Thus, the last 2 events `{"CreatedTime": "2019-02-05 10:14:00", "Type": "SELL", "Amount": 300, "BrokerCode": "ABX"}` and `{"CreatedTime": "2019-02-05 10:16:00", "Type": "SELL", "Amount": 300, "BrokerCode": "ABX"}` will be ignored by Spark.

Key takeaways
- Watermark is the key for state store cleanup
- Events within the watermark is taken
- Events outside the watermark may or may not be taken, depends on whether the windows state is cleanup or not.

# Watermark and output modes
State store cleanup
1. Setting a watermark
2. Using an appropriate output mode

Output Modes
1. Complete - State store kept forever, use with caution for aggregation. Watermark does not take effect here.
2. Update - Upsert, produce new aggregates, emit those aggregates which are updated or changed. This is the most useful and efficient mode for streaming queries with streaming aggregates. This mode shouldn't be used with append only sinks such as file sink, it will result in duplicate records as Spark always create new files.
3. Append - It suppress the output of the windowing aggregates until they cross the watermark boundary.

# Sliding Window
Almost the same as tumbling window expect the sliding window allows overlapping windows. In `window(col("CreatedTime"), "15 minute", "5 minute"))` the 3rd argument is the slide duration.

# Joining Stream to static source (Cassandra database)
To use cassandra, `.config("spark.jars.packages", "org.apache.spark:spark-sql-kafka-0-10_2.12:3.0.1,org.apache.spark:spark-avro_2.12:3.0.1,com.datastax.spark:spark-cassandra-connector_2.12:3.0.0-beta"` is needed for Spark config.

There is no streaming cassandra sink, but cassandra connector could be used to handle the write of each micro batch using a customized function.
```
output_df.writeStream \
    .foreachBatch(write_to_cassandra)
```

# Joining Stream to another Stream, Streaming Watermark
When stream joins with stream, every entry from both streams is kept in the state store. Watermark is needed to avoid OOM.

Pay a little attention when designing the event model, as duplicates could happen if the joining key is not unique.

# Streaming Outer Joins
1. Left Outer - Left side must be a stream
2. Right Outer - Right side must be a stream
3. Full Outer - Not allowed

Outer joins in Spark streaming (a workaround)
- Left Outer
  - Watermark on the right-side stream
  - Maximum time range constraint between the left and right-side events
- Right Outer
  - Watermark on the left-side stream
  - Maximum time range constraint between the left and right-side events

The above mentioned "time range constraint" means the time delay between the impression and click events. It's implemented in the code as below condition. The unmatched records are sent only in the next micro-batch after the watermark expires.
```
join_expr = "ImpressionID == ClickID" + \
    " AND ClickTime BETWEEN ImpressionTime AND ImpressionTime + interval 15 minute"
```