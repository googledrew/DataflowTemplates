# PubSub CDC to Redis Dataflow Template

The [PubSubCdcToRedis](src/main/java/com/google/cloud/teleport/v2/templates/PubSubCdcToRedis.java) pipeline
ingests Striim CDC changes from Pub/Sub subscription and writes the records in Cloud Redis database.

This Dataflow pipeline expects:
- Cloud Redis database and tables to be pre-created with the correct schema.
- Pub/Sub subscription contains CDC changes from the same database type (e.g. Oracle SAP). Currently only tested against [Oracle DB](https://www.striim.com/docs/en/oracle-reader-and-ojet-waevent-fields.html) changes.

## Getting Started

### Requirements
* Java 8
* Maven
* Striim CDC changes in Pub/Sub subscription
* Cloud Redis database pre-created
* GCS Bucket as DLQ for failed messages

### Building Template
This is a Flex Template meaning that the pipeline code will be containerized and the container will be
used to launch the Dataflow pipeline.

#### Building Dependencies

Run `mvn install` in `common` directory, then in this directory `v2/pubsub-cdc-to-redis`

#### Building Container Image
* Set environment variables.

```sh
export PROJECT=<my-project>
export IMAGE_NAME=datastream-to-redis
export BUCKET_NAME=gs://<bucket-name>
export TARGET_GCR_IMAGE=gcr.io/${PROJECT}/${IMAGE_NAME}
export BASE_CONTAINER_IMAGE=gcr.io/dataflow-templates-base/java8-template-launcher-base
export BASE_CONTAINER_IMAGE_VERSION=latest
export APP_ROOT=/template/${IMAGE_NAME}
export DATAFLOW_JAVA_COMMAND_SPEC=${APP_ROOT}/resources/${IMAGE_NAME}-command-spec.json
export TEMPLATE_IMAGE_SPEC=${BUCKET_NAME}/images/${IMAGE_NAME}-image-spec.json

export INPUT_SUBSCRIPTION=projects/${PROJECT}/subscriptions/<subscription-name>
export DEADLETTER_QUEUE_DIR=gs://${BUCKET_NAME}/dlq
export REDIS_IP_ADDRESS=<redis-ip-address>
export REDIS_PORT=<redis-port>

gcloud config set project ${PROJECT}
```

* Build and push image to Google Container Repository

```sh
mvn clean package \
-Dimage=${TARGET_GCR_IMAGE} \
-Dbase-container-image=${BASE_CONTAINER_IMAGE} \
-Dbase-container-image.version=${BASE_CONTAINER_IMAGE_VERSION} \
-Dapp-root=${APP_ROOT} \
-Dcommand-spec=${DATAFLOW_JAVA_COMMAND_SPEC} \
-am -pl ${IMAGE_NAME}
```

### Testing Template

The template unit tests can be run using:
```sh
mvn test
```

### Executing Template

The template requires the following parameters:
* inputSubscription: The Pub/Sub subscriptiom which contains Striim's CDC changes for a given database.
* redisIpAddress: The Cloud Redis ip address where the stream must be replicated.
* redisPort: The Cloud Redis port where the stream must be replicated.
* outputDeadletterTable: Deadletter table for failed inserts.

Template can be executed using the following API call:
```sh
export JOB_NAME="${IMAGE_NAME}-`date +%Y%m%d-%H%M%S-%N`"
gcloud beta dataflow flex-template run ${JOB_NAME} \
        --project=${PROJECT} --region=us-central1 \
        --template-file-gcs-location=${TEMPLATE_IMAGE_SPEC} \
        --parameters redisIpAddress=${REDIS_IP_ADDRESS},redisPort=${REDIS_PORT},inputSubscription=${INPUT_SUBSCRIPTION},deadLetterQueueDirectory=${DEADLETTER_QUEUE_DIR}
```

Alternatively, using Maven:

```
mvn compile exec:java -Dexec.mainClass=com.google.cloud.teleport.v2.templates.PubSubCdcToRedis -Dexec.cleanupDaemonThreads=false -Dexec.args=" \
--project=${PROJECT} \
--stagingLocation=gs://${BUCKET_NAME}/staging \
--tempLocation=gs://${BUCKET_NAME}/temp \
--inputSubscription=${INPUT_SUBSCRIPTION} \
--deadLetterQueueDirectory=${DEADLETTER_QUEUE_DIR} \
--redisIpAddress=${REDIS_IP_ADDRESS} \
--redisPort=${REDIS_PORT}" \
```
