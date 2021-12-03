# Kafka on Red Hat OpenShift Demo
This demo showcases various capabilities of running Kafka on OpenShift.

Before any of these, be sure to install the **Red Hat Integration - AMQ Streams Operator** from OperatorHub into **All Namespaces**.
## Basic Installation
1. From the **Developer Perspective**, click **+Add**, and select **Operator Backed** under **Developer Catalog**.
2. Select **Kafka**, click **Create**, and install with defaults. Optionally browse the Form view here to demonstrate the configuration options that are available.
3. When Kafka comes up, create a topic and send some messages:
    ```bash
    oc exec -it my-cluster-kafka-0 -- /opt/kafka/bin/kafka-console-producer.sh \
        --bootstrap-server localhost:9092 \
        --topic my-topic
    ```
4. In a separate console, open a consumer to read them:
    ```bash
    oc exec -it my-cluster-kafka-0 -- /opt/kafka/bin/kafka-console-consumer.sh \
        --bootstrap-server localhost:9092 \
        --topic my-topic \
        --from-beginning
    ```
5. Delete one of the broker pods to demonstrate auto-healing capability.
6. Increase the number of replicas to show auto-scaling ability.

## Working with Authorization
Create a `Kafka` instance:
* Kafka -> Listeners -> plain -> Authentication -> Type = scram-sha-512
* Kafka -> Authorization -> Type = simple

> Alternatively, just use the CR in this repo: `oc apply -f kafka.yml`


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


## Working with a Service Registry
I **highly** recommend deploying [this excellent demo](https://github.com/rmarting/kafka-clients-quarkus-sample) ahead of time, and walking through the different connections.

> You may need to update a few `apiVersion` tags to differentiate from upstream, and `oc expose svc kafka-clients-quarkus-sample` may be required to expose a `Route` for the quarkus app.

## Working with Kafka Connect
Consider leveraging [a demo I built](https://github.com/andykrohg/db2-debezium) to demonstrate **Kafka Connect** and **Debezium** for zero-code streaming pipelines.