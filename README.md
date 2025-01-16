# Fraud Detection with Kafka Streams 
This project aims to develop a dashboard using Grafana for visualizing data stored in an InfluxDB bucket named fraud-detection. The focus is on providing clear and interactive visualizations of transaction data, including amounts and user activities, to facilitate real-time monitoring and analysis without applying data aggregation.
## Key Features 
* Real-time Transaction Processing 
* Storage of Suspicious Transactions
* Data Visualization
* Simplified Deployment
## Architecture
The overall project architecture is illustrated below:

* Kafka Streams reads raw transactions from the transactions-input topic.
* Kafka Streams applies fraud detection logic.
* Suspicious transactions are published to the fraud-alerts topic.
* These transactions are logged in InfluxDB.
* Grafana fetches data from InfluxDB for visualization.

## Setup of Kafka cluster
We configured a Docker Compose environment to run our Kafka cluster. The configuration includes a Zookeeper service and a Kafka broker. 
This setup ensures a single-node Kafka cluster suitable for development and testing.
```yaml
version: '3.8'
networks:
  monitoring:
    driver: bridge
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    networks:
      - monitoring
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:latest
    networks:
      - monitoring
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  influxdb:
    image: influxdb:latest
    networks:
      - monitoring
    ports:
      - "8086:8086"
    environment:
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME: admin
      DOCKER_INFLUXDB_INIT_PASSWORD: adminpassword
      DOCKER_INFLUXDB_INIT_ORG: enset
      DOCKER_INFLUXDB_INIT_BUCKET: fraud_transaction
      DOCKER_INFLUXDB_INIT_ADMIN_TOKEN: secret-token

  grafana:
    image: grafana/grafana:latest
    networks:
      - monitoring
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin
    depends_on:
      - influxdb
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  grafana-data:
```

![dockerRun.png](screenshots%2FdockerRun.png)

## Implementation
1. Transaction Class
This class represents a financial transaction through three attributes: userId (user identifier), amount (transaction amount), and timestamp (transaction timestamp). It is designed for use in applications that require processing or exchanging transaction-related data, particularly via JSON.
``` java
package org.example;

import com.fasterxml.jackson.annotation.JsonProperty;

public class Transaction {
    private String userId;
    private double amount;
    private int timestamp;

    public Transaction() {}

    public Transaction(String userId, double amount, int timestamp) {
        this.userId = userId;
        this.amount = amount;
        this.timestamp = timestamp;
    }

    @JsonProperty("userId")
    public String getUserId() {
        return userId;
    }

    @JsonProperty("amount")
    public double getAmount() {
        return amount;
    }

    @JsonProperty("timestamp")
    public int getTimestamp() {
        return timestamp;
    }

    public String toString() {
        return "Transaction{" +
                "userId='" + userId + '\'' +
                ", amount=" + amount +
                ", timestamp=" + timestamp +
                '}';
    }
}
```
2.  TransactionProducer Class
The `TransactionProducer` class is a Kafka producer that generates random transactions and sends them to the Kafka topic `transactions-input`. It configures the producer with essential properties, such as the Kafka server address (`localhost:9092`) and string serializers for keys and values. In each iteration, it generates a transaction using the `generateTransaction()` method, which creates a unique `userId`, a random `amount`, and a current `timestamp`. The transaction is serialized to JSON using Jackson's `ObjectMapper` and sent to Kafka as a message. The producer operates continuously, sending messages every second and handling errors during transmission.
``` java
package org.example;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.kafka.common.serialization.StringSerializer;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.producer.ProducerRecord;

import java.util.Properties;
import java.util.Random;

public class TransactionProducer {

    private static final String TOPIC = "transactions-input";
    private static final Random RANDOM = new Random();
    private static final ObjectMapper MAPPER = new ObjectMapper();

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

        try (KafkaProducer<String, String> producer = new KafkaProducer<>(props)) {
            while (true) {
                Transaction transaction = generateTransaction();
                String json = MAPPER.writeValueAsString(transaction);

                ProducerRecord<String, String> record =
                        new ProducerRecord<>(TOPIC, transaction.getUserId(), json);

                producer.send(record, (metadata, exception) -> {
                    if (exception != null) {
                        System.err.println("Error sending message: " + exception.getMessage());
                    } else {
                        System.out.println("Sent transaction: " + json);
                    }
                });

                Thread.sleep(1000);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    private static Transaction generateTransaction() {
        String userId = String.format("user_%03d", RANDOM.nextInt(200));
        double amount = 1000 + RANDOM.nextDouble() * 10000;
        int timestamp = (int) (System.currentTimeMillis() / 1000);
        return new Transaction(userId, amount, timestamp);
    }
}
```
3.  TransactionProcessor Class
The `TransactionProcessor` class is a Kafka Streams processor that receives transactions from the `transactions-input` topic, analyzes them, and generates fraud alerts for suspicious transactions. The program configures the Kafka Streams environment with necessary properties, such as the application ID, Kafka server, and default serializers for keys and values. A stream is created from the `transactions-input` topic, and each message is deserialized into a `Transaction` object.
The stream is then branched using the `branch` method: one branch for transactions above a suspicious threshold (10,000.00) and another for the rest. Suspicious transactions are serialized to JSON and sent to the `fraud-alerts` topic, while also being logged to the console for alerts. Finally, the stream is started with a shutdown hook to ensure a clean termination when the program stops.
```java
package org.example;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.streams.kstream.KStream;

import java.util.Properties;

public class TransactionProcessor {
    private static final String INPUT_TOPIC = "transactions-input";
    private static final String OUTPUT_TOPIC = "fraud-alerts";
    private static final double SUSPICIOUS_AMOUNT = 10_000.0;
    private static final ObjectMapper MAPPER = new ObjectMapper();

    public static void main(String[] args) {
        Properties config = new Properties();
        config.put(StreamsConfig.APPLICATION_ID_CONFIG, "transaction-processor");
        config.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        config.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        StreamsBuilder builder = new StreamsBuilder();

        KStream<String, String> inputStream = builder.stream(INPUT_TOPIC);
        KStream<String, Transaction>[] branches = inputStream
                .mapValues(value -> {
                    try {
                        return MAPPER.readValue(value, Transaction.class);
                    } catch (Exception e) {
                        System.err.println("Error parsing transaction: " + e.getMessage());
                        return null;
                    }
                })
                .filter((key, transaction) -> transaction != null)
                .branch(
                        (key, transaction) -> transaction.getAmount() > SUSPICIOUS_AMOUNT,
                        (key, transaction) -> true
                );

        branches[0]
                .mapValues(transaction -> {
                    try {
                        return MAPPER.writeValueAsString(transaction);
                    } catch (Exception e) {
                        System.err.println("Error serializing transaction: " + e.getMessage());
                        return null;
                    }
                })
                .filter((key, json) -> json != null)
                .peek((key, json) -> System.out.println("Fraud alert: " + json))
                .to(OUTPUT_TOPIC);

        KafkaStreams streams = new KafkaStreams(builder.build(), config);
        streams.start();

        // Shutdown hook
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
}
```
4. FraudAlertConsumer Class
The `FraudAlertConsumer` class is a Kafka consumer that retrieves fraud alerts from the `fraud-alerts` topic and stores them in an InfluxDB database. It begins by loading necessary properties from an `application.properties` file to connect to InfluxDB, including the URL and access token. The consumer is then configured to connect to `localhost:9092`, subscribe to the `fraud-alerts` topic, and use string deserializers to process incoming messages.
Each received message (a fraud alert) is deserialized into a `Transaction` object. A measurement point is created in InfluxDB, where the `userId`, `amount`, and `timestamp` of the transaction are stored as fields and tags. The alerts are stored in the `fraud_transaction` bucket of InfluxDB with second-level precision.
```java
package org.example;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.kafka.common.serialization.StringDeserializer;
import com.influxdb.client.InfluxDBClient;
import com.influxdb.client.InfluxDBClientFactory;
import com.influxdb.client.WriteApi;
import com.influxdb.client.write.Point;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import com.influxdb.client.domain.WritePrecision;

import java.io.FileInputStream;
import java.io.IOException;
import java.time.Duration;
import java.util.Collections;
import java.util.Properties;

public class FraudAlertConsumer {
    private static final String TOPIC = "fraud-alerts";
    private static final String INFLUXDB_ORG = "enset";
    private static final String INFLUXDB_BUCKET = "fraud_transaction";
    private static final ObjectMapper MAPPER = new ObjectMapper();

    public static void main(String[] args) {
        Properties appProps = new Properties();
        try (FileInputStream fis = new FileInputStream("src/main/resources/application.properties")) {
            appProps.load(fis);
        } catch (IOException e) {
            System.err.println("Error loading properties file: " + e.getMessage());
            return;
        }

        String INFLUXDB_URL = appProps.getProperty("influxdb.url");
        String INFLUXDB_TOKEN = appProps.getProperty("influxdb.token");
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "fraud-alert-consumer");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        InfluxDBClient influxDBClient = InfluxDBClientFactory.create(
                INFLUXDB_URL,
                INFLUXDB_TOKEN.toCharArray(),
                INFLUXDB_ORG,
                INFLUXDB_BUCKET
        );

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
             WriteApi writeApi = influxDBClient.getWriteApi()) {

            consumer.subscribe(Collections.singletonList(TOPIC));
            System.out.println("Consuming fraud alerts...");

            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

                records.forEach(record -> {
                    try {
                        Transaction transaction = MAPPER.readValue(record.value(), Transaction.class);

                        Point point = Point.measurement("fraud")
                                .addTag("userId", transaction.getUserId())
                                .addField("amount", transaction.getAmount())
                                .time(transaction.getTimestamp(), WritePrecision.S);

                        writeApi.writePoint(point);

                        System.out.println("Stored fraud alert: " + transaction);
                    } catch (Exception e) {
                        System.err.println("Error processing record: " + e.getMessage());
                    }
                });
            }
        } catch (Exception e) {
            System.err.println("Error in consumer: " + e.getMessage());
        } finally {
            influxDBClient.close();
        }
    }
}

```
## Results
* Topics creation:
![img.png](screenshots%2Fimg.png)
* Transaction Producing:
![sentTransactions.jpg](screenshots%2FsentTransactions.jpg) 
* Transaction processing:
![FraudAlerts.jpg](screenshots%2FFraudAlerts.jpg)
* Stored Fraud Alerts:
![storedFraudAlerts.jpg](screenshots%2FstoredFraudAlerts.jpg)
* Visualisation of the fraud alerts in a table:
![raudTable.jpg](screenshots%2FraudTable.jpg)
* Fraudulent transactions and their amounts over time.
![AmountHistogram.jpg](screenshots%2FAmountHistogram.jpg)
* This dashboard monitors fraudulent transactions, showcasing the top 10 users with the highest fraud amounts, key real-time stats (minimum, maximum, total amounts, and transaction count), and a table of recent fraud transactions with timestamps, amounts, and user IDs, enabling quick detection and analysis of fraud patterns.
![top10scammers.jpg](screenshots%2Ftop10scammers.jpg)