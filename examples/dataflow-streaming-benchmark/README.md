# Dataflow Streaming Benchmark

When developing Dataflow pipelines, it's common to want to benchmark them at a specific QPS using
fake or generated data. This pipeline takes in a QPS parameter, a path to a schema file, and 
generates fake JSON messages matching the schema to a Pub/Sub topic at the rate of the QPS.

## Pipeline

[StreamingBenchmark](src/main/java/com/google/cloud/pso/pipeline/StreamingBenchmark.java) -
A streaming pipeline which generates messages at a specified rate to a Pub/Sub topic. The messages 
are generated according to a schema template which instructs the pipeline how to populate the 
messages with fake data compliant to constraints.

> Note the number of workers executing the pipeline must be large enough to support the supplied 
> QPS. Use a general rule of 2,500 QPS per core in the worker pool when configuring your pipeline.


![Pipeline DAG](img/pipeline-dag.png "Pipeline DAG")

## Getting Started

### Requirements

* Java 8
* Maven 3

### Building the Project

Build the entire project using the maven compile command.
```sh
mvn clean compile
```

### Creating the Schema File
The schema file used to generate JSON messages with fake data is based on the 
[json-data-generator](https://github.com/vincentrussell/json-data-generator) library. This library
allows for the structuring of a sample JSON schema and injection of common faker functions to 
instruct the data generator of what type of fake data to create in each field. See the 
json-data-generator [docs](https://github.com/vincentrussell/json-data-generator) for more 
information on the faker functions.

#### Example Schema File
Below is an example schema file which generates fake game event payloads with random data.
```javascript
{
  "eventId": "{{uuid()}}",
  "eventTimestamp": {{timestamp()}},
  "ipv4": "{{ipv4()}}",
  "ipv6": "{{ipv6()}}",
  "country": "{{country()}}",
  "username": "{{username()}}",
  "quest": "{{random("A Break In the Ice", "Ghosts of Perdition", "Survive the Low Road")}}",
  "score": {{integer(100, 10000)}},
  "completed": {{bool()}}
}
```

#### Example Output Data
Based on the above schema, the below would be an example of a message which would be output to the
Pub/Sub topic.
```javascript
{
  "eventId": "5dacca34-163b-42cb-872e-fe3bad7bffa9",
  "eventTimestamp": 1537729128894,
  "ipv4": "164.215.241.55",
  "ipv6": "e401:58fc:93c5:689b:4401:206f:4734:2740",
  "country": "Montserrat",
  "username": "asellers",
  "quest": "A Break In the Ice",
  "score": 2721,
  "completed": false
}
```

### Executing the Pipeline
```bash
# Change to project directory
cd examples/dataflow-streaming-benchmark/

#Set authetication
export GOOGLE_APPLICATION_CREDENTIALS="/Users/khanz/dataflow-learning-01bcff791ed1.json"

# Set the pipeline vars
PROJECT_ID=dataflow-learning-khanz
BUCKET=dataflow-learning-khanz
PIPELINE_FOLDER=gs://${BUCKET}/dataflow/pipelines/streaming-benchmark
SCHEMA_LOCATION=gs://dataflow-learning-khanz/schema.json
PUBSUB_TOPIC=projects/$PROJECT_ID/topics/test-topic
QPS=5
RUNNER=DataflowRunner
ZONE=us-west1-a

#Create a pub/sub topic
gcloud pubsub topics create $PUBSUB_TOPIC

# Build the template
mvn compile exec:java \
-Dexec.mainClass=com.google.cloud.pso.pipeline.StreamingBenchmark \
-Dexec.cleanupDaemonThreads=false \
-Dexec.args=" \
    --project=${PROJECT_ID} \
    --stagingLocation=${PIPELINE_FOLDER}/staging \
    --tempLocation=${PIPELINE_FOLDER}/temp \
    --runner=${RUNNER} \
    --zone=${ZONE} \
    --autoscalingAlgorithm=THROUGHPUT_BASED \
    --maxNumWorkers=5 \
    --qps=${QPS} \
    --schemaLocation=${SCHEMA_LOCATION} \
    --topic=${PUBSUB_TOPIC}"

# Create a Subscription
gcloud pubsub subscriptions create test-sub --topic=$PUBSUB_TOPIC --topic-project=$PROJECT_ID

#Check messages from the pub/sub
gcloud pubsub subscriptions pull --auto-ack projects/dataflow-learning-khanz/subscriptions/test-sub
```
