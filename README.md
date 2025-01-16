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
[docker-compose.yml](docker-compose.yml)

![dockerRun.png](screenshots%2FdockerRun.png)

## Implementation
1. Transaction Class
This class represents a financial transaction through three attributes: userId (user identifier), amount (transaction amount), and timestamp (transaction timestamp). It is designed for use in applications that require processing or exchanging transaction-related data, particularly via JSON.
[Transaction.java](src%2Fmain%2Fjava%2Forg%2Fexample%2FTransaction.java)
2.  TransactionProducer Class
The `TransactionProducer` class is a Kafka producer that generates random transactions and sends them to the Kafka topic `transactions-input`. It configures the producer with essential properties, such as the Kafka server address (`localhost:9092`) and string serializers for keys and values. In each iteration, it generates a transaction using the `generateTransaction()` method, which creates a unique `userId`, a random `amount`, and a current `timestamp`. The transaction is serialized to JSON using Jackson's `ObjectMapper` and sent to Kafka as a message. The producer operates continuously, sending messages every second and handling errors during transmission.
[TransactionProducer.java](src%2Fmain%2Fjava%2Forg%2Fexample%2FTransactionProducer.java)
3.  TransactionProcessor Class
The `TransactionProcessor` class is a Kafka Streams processor that receives transactions from the `transactions-input` topic, analyzes them, and generates fraud alerts for suspicious transactions. The program configures the Kafka Streams environment with necessary properties, such as the application ID, Kafka server, and default serializers for keys and values. A stream is created from the `transactions-input` topic, and each message is deserialized into a `Transaction` object.
The stream is then branched using the `branch` method: one branch for transactions above a suspicious threshold (10,000.00) and another for the rest. Suspicious transactions are serialized to JSON and sent to the `fraud-alerts` topic, while also being logged to the console for alerts. Finally, the stream is started with a shutdown hook to ensure a clean termination when the program stops.
[TransactionProcessor.java](src%2Fmain%2Fjava%2Forg%2Fexample%2FTransactionProcessor.java)
4. FraudAlertConsumer Class
The `FraudAlertConsumer` class is a Kafka consumer that retrieves fraud alerts from the `fraud-alerts` topic and stores them in an InfluxDB database. It begins by loading necessary properties from an `application.properties` file to connect to InfluxDB, including the URL and access token. The consumer is then configured to connect to `localhost:9092`, subscribe to the `fraud-alerts` topic, and use string deserializers to process incoming messages.
Each received message (a fraud alert) is deserialized into a `Transaction` object. A measurement point is created in InfluxDB, where the `userId`, `amount`, and `timestamp` of the transaction are stored as fields and tags. The alerts are stored in the `fraud_transaction` bucket of InfluxDB with second-level precision.
[FraudAlertConsumer.java](src%2Fmain%2Fjava%2Forg%2Fexample%2FFraudAlertConsumer.java)

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