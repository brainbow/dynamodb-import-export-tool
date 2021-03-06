# DynamoDB Import Export Tool

## Running as an executable

mvn exec:java -Dexec.mainClass="com.amazonaws.dynamodb.bootstrap.CommandLineInterface" \

 -Dexec.args=" \
 
 --destinationEndpoint dynamodb.us-east-1.amazonaws.com \
 
 --destinationTable nameOfDestinationTable  \
 
 --sourceEndpoint dynamodb.us-west-1.amazonaws.com \
 
 --sourceTable nameOfSourceTable \
 
 --readThroughputRatio .5  \
 
 --writeThroughputRatio .7 \
 
 --maxWriteThreads 128 \
 
 --totalSections 4 \
 
 --section 3"

### Description of Arguments

destinationEndpoint: The endpoint where the destination table is located

destinationTable: The destination table to write to

sourceEndpoint: The endpoint where the source table is located

sourceTable: The source table to read from

readThroughputRatio: The ratio of read throughput to consume from the source table

writeThroughputRatio: The ratio of write throughput to consume from the destination table

maxWriteThreads: (Optional, default=128 * Available_Processors) Maximum number of write threads to create

totalSections: (Optional, default=1) Total number of sections to split the bootstrap into. Each application will only scan and write one section

section: (Optional, default=0) section to read and write. Only will scan this one section of all sections, [0...totalSections-1]
 

## Using the API

### 1. Transfer Data from One DynamoDB Table to Another DynamoDB Table

The below example will read from "mySourceTable" at 100 reads per second, using 4 threads. And it will write to "myDestinationTable" at 50 writes per second, using 8 threads.
Both tables are located at "dynamodb.us-west-1.amazonaws.com". (to transfer to a different region, create 2 AmazonDynamoDBClients
with different endpoints to pass into the DynamoDBBootstrapWorker and the DynamoDBConsumer.

```java
AmazonDynamoDBClient client = new AmazonDynamoDBClient(new ProfileCredentialsProvider());
client.setEndpoint("dynamodb.us-west-1.amazonaws.com");

DynamoDBBootstrapWorker worker = null;

try {
    // 100.0 read operations per second. 4 threads to scan the table.
    worker = new DynamoDBBootstrapWorker(client,
                100.0, "mySourceTable", 4);
} catch (NullReadCapacityException e) {
    LOGGER.error("The DynamoDB source table returned a null read capacity.", e);
    System.exit(1);
}

 // 50.0 write operations per second. 8 threads to scan the table.
DynamoDBConsumer consumer = new DynamoDBConsumer(client, "myDestinationTable", 50.0, Executors.newFixedThreadPool(8));

try {
    worker.pipe(consumer);
} catch (ExecutionException e) {
    LOGGER.error("Encountered exception when executing transfer.", e);
    System.exit(1);
} catch (InterruptedException e){
    LOGGER.error("Interrupted when executing transfer.", e);
    System.exit(1);
}
```


### 2. Transfer Data From one DynamoDB Table to a Blocking Queue.

The below example will read from a DynamoDB table and export to an array blocking queue. This is useful for when another application would like to consume
the DynamoDB entries but does not have a setup application for it. They can just retrieve the queue (consumer.getQueue()) and then continually pop() from it
to then process the new entries.

```java
AmazonDynamoDBClient client = new AmazonDynamoDBClient(new ProfileCredentialsProvider());
client.setEndpoint("dynamodb.us-west-1.amazonaws.com");

DynamoDBBootstrapWorker worker = null;

try {
    // 100.0 read operations per second. 4 threads to scan the table.
    worker = new DynamoDBBootstrapWorker(client,
                100.0, "mySourceTable", 4);
} catch (NullReadCapacityException e) {
    LOGGER.error("The DynamoDB source table returned a null read capacity.", e);
    System.exit(1);
}

BlockingQueueConsumer consumer = new BlockingQueueConsumer(8);

try {
    worker.pipe(consumer);
} catch (ExecutionException e) {
    LOGGER.error("Encountered exception when executing transfer.", e);
    System.exit(1);
} catch (InterruptedException e){
    LOGGER.error("Interrupted when executing transfer.", e);
    System.exit(1);
}
```
