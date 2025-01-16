#Fraud Detection with Kafka Streams 
This project aims to develop a dashboard using Grafana for visualizing data stored in an InfluxDB bucket named fraud-detection. The focus is on providing clear and interactive visualizations of transaction data, including amounts and user activities, to facilitate real-time monitoring and analysis without applying data aggregation.
###Key Features
1. Real-time Transaction Processing
2. Storage of Suspicious Transactions
3. Data Visualization
4. Simplified Deployment
###Architecture
The overall project architecture is illustrated below:

Kafka Streams reads raw transactions from the transactions-input topic.
Kafka Streams applies fraud detection logic.
Suspicious transactions are published to the fraud-alerts topic.
These transactions are logged in InfluxDB.
Grafana fetches data from InfluxDB for visualization.

###Setup of Kafka cluster
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