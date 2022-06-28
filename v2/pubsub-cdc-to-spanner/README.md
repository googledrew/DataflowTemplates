# DataStream to Spanner Dataflow Template

The [PubSubCdcToSpanner](src/main/java/com/google/cloud/teleport/v2/templates/PubSubCdcToSpanner.java) pipeline
ingests Striim CDC changes from Pub/Sub subscription and writes the records in Cloud Spanner database.

This Dataflow pipeline expects:
- Cloud Spanner database and tables to be pre-created with the correct schema.
- Pub/Sub subscription contains CDC changes from the same database type (e.g. Oracle SAP). Currently only tested against [Oracle DB](https://www.striim.com/docs/en/oracle-reader-and-ojet-waevent-fields.html) changes.

## Getting Started

### Requirements
* Java 8
* Maven
* Striim CDC changes in Pub/Sub subscription
* Cloud Spanner database and tables pre-created
* GCS Bucket as DLQ for failed messages

### Building Template
This is a Flex Template meaning that the pipeline code will be containerized and the container will be
used to launch the Dataflow pipeline.

#### Building Dependencies

Run `mvn install` in `common` directory, then in this directory `v2/pubsub-cdc-to-spanner`

#### Building Container Image
* Set environment variables.

```sh
export PROJECT=<my-project>
export IMAGE_NAME=datastream-to-spanner
export BUCKET_NAME=gs://<bucket-name>
export TARGET_GCR_IMAGE=gcr.io/${PROJECT}/${IMAGE_NAME}
export BASE_CONTAINER_IMAGE=gcr.io/dataflow-templates-base/java8-template-launcher-base
export BASE_CONTAINER_IMAGE_VERSION=latest
export APP_ROOT=/template/${IMAGE_NAME}
export DATAFLOW_JAVA_COMMAND_SPEC=${APP_ROOT}/resources/${IMAGE_NAME}-command-spec.json
export TEMPLATE_IMAGE_SPEC=${BUCKET_NAME}/images/${IMAGE_NAME}-image-spec.json

export INPUT_SUBSCRIPTION=projects/${PROJECT}/subscriptions/<subscription-name>
export DEADLETTER_QUEUE_DIR=gs://${BUCKET_NAME}/dlq
export INSTANCE_ID=<spanner-instance-id>
export DATABASE_ID=<spanner-database-id>

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
* instanceId: The Cloud Spanner instance id where the stream must be replicated.
* databaseId: The Cloud Spanner database id where the stream must be replicated.
* outputDeadletterTable: Deadletter table for failed inserts.

Template can be executed using the following API call:
```sh
export JOB_NAME="${IMAGE_NAME}-`date +%Y%m%d-%H%M%S-%N`"
gcloud beta dataflow flex-template run ${JOB_NAME} \
        --project=${PROJECT} --region=us-central1 \
        --template-file-gcs-location=${TEMPLATE_IMAGE_SPEC} \
        --parameters instanceId=${INSTANCE_ID},databaseId=${DATABASE_ID},inputSubscription=${INPUT_SUBSCRIPTION},deadLetterQueueDirectory=${DEADLETTER_QUEUE_DIR}
```

Alternatively, using Maven:

```
mvn compile exec:java -Dexec.mainClass=com.google.cloud.teleport.v2.templates.PubSubCdcToSpanner -Dexec.cleanupDaemonThreads=false -Dexec.args=" \
--project=${PROJECT} \
--stagingLocation=gs://${BUCKET_NAME}/staging \
--tempLocation=gs://${BUCKET_NAME}/temp \
--inputSubscription=${INPUT_SUBSCRIPTION} \
--deadLetterQueueDirectory=${DEADLETTER_QUEUE_DIR} \
--instanceId=${INSTANCE_ID} \
--databaseId=${DATABASE_ID}" \
```
