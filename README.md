# Kafka on Red Hat OpenShift Demo
This demo showcases various capabilities of running Kafka on OpenShift.

## Installing Kafka
Here we'll create a Kafka cluster with accompanying ZooKeeper cluster 

1. Install the **Red Hat Integration - AMQ Streams Operator** from OperatorHub.
2. Create a `Kafka` instance using the CR in this repo:
    ```bash
    oc apply -f kafka.yml
    ```

## Creating Users
1. Create two `KafkaUsers`: **michael** and **dwight**. Michael has `All` privileges on the topic **scranton**, whereas Dwight only has `Read` and `Describe` access.
    ```bash
    oc apply -f kafka-user-michael.yml -f kafka-user-dwight.yml
    ```
2. Wait for them to be ready, then grab the passwords the operator creates for them:
    ```bash
    # michael
    oc wait kafkauser/michael --for=condition=ready
    MICHAEL_PASSWORD=$(oc extract secret/michael --keys=password --to=-)

    # dwight
    oc wait kafkauser/dwight --for=condition=ready
    DWIGHT_PASSWORD=$(oc extract secret/dwight --keys=password --to=-)
    ```

## Connecting to Kafka
1. As `michael` produce some messages to the `scranton` topic:
    ```bash
    oc exec -it my-cluster-kafka-0 -- /opt/kafka/bin/kafka-console-producer.sh \
        --bootstrap-server localhost:9092 \
        --topic scranton \
        --producer-property sasl.mechanism=SCRAM-SHA-512 \
        --producer-property security.protocol=SASL_PLAINTEXT \
        --producer-property "sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
            username=michael \
            password=$MICHAEL_PASSWORD ;"
    ```
2. Then read them as `michael`:
    ```bash
    oc exec -it my-cluster-kafka-0 -- /opt/kafka/bin/kafka-console-consumer.sh \
        --bootstrap-server localhost:9092 \
        --topic scranton \
        --from-beginning \
        --consumer-property sasl.mechanism=SCRAM-SHA-512 \
        --consumer-property security.protocol=SASL_PLAINTEXT \
        --consumer-property "sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
            username=michael \
            password=$MICHAEL_PASSWORD ;"
    ```
3. Try to produce a message as `dwight`:
    ```bash
    oc exec -it my-cluster-kafka-0 -- /opt/kafka/bin/kafka-console-producer.sh \
        --bootstrap-server localhost:9092 \
        --topic scranton \
        --producer-property sasl.mechanism=SCRAM-SHA-512 \
        --producer-property security.protocol=SASL_PLAINTEXT \
        --producer-property "sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
            username=dwight \
            password=$DWIGHT_PASSWORD ;"
    ```
    > You should receive the error `Not authorized to access topics: [scranton]`, since `dwight` has read-only access.
4. Then try to consume as `dwight`:
    ```bash
    oc exec -it my-cluster-kafka-0 -- /opt/kafka/bin/kafka-console-consumer.sh \
        --bootstrap-server localhost:9092 \
        --topic scranton \
        --from-beginning \
        --consumer-property sasl.mechanism=SCRAM-SHA-512 \
        --consumer-property security.protocol=SASL_PLAINTEXT \
        --consumer-property "sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
            username=dwight \
            password=$DWIGHT_PASSWORD ;"
    ```


## Deploying a Service Registry
Now we'll deploy a Service Registry to manage topic schemas.

1. Install the **Red Hat Integration - Service Registry Operator** from OperatorHub.
2. Next we'll create an `ApicurioRegistry`, using the existing Kafka cluster as a data store. The Service Registry will need elevated privileges to Kafka to manage its own topics, so create a `KafkaUser` for it to use:
    ```bash
    oc apply -f service-registry/kafka-user-registry.yml
    ```
3. Then create the registry itself:
    ```bash
    oc apply -f service-registry/service-registry.yml
    ```
4. Open the Route for Apicurio Registry to view the dashboard.
5. Click **Upload Artifact**, and either drag/drop or paste the content from `service-registry/fullname-schema.json`