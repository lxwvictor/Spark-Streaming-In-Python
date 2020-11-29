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

## Start zookeeper, then kafka server
    zookeeper-server-start.bat $KAFKA_HOME/config/zookeeper.properties
    kafka-server-start.bat $KAFKA_HOME/config/server.properties

## Stop kafka server and zookeeper
    kafka-server-stop.bat
    zookeeper-server-stop.bat

## Create topic
    kafka-topics.bat --create --topic quickstart-events --bootstrap-server localhost:9092

## Start consumer
    kafka-console-consumer.bat --topic quickstart-events --from-beginning --bootstrap-server localhost:9092

## Start producer
    kafka-console-producer.bat --topic quickstart-events --bootstrap-server localhost:9092
    hello
    hi