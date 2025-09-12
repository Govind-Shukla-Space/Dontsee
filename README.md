# 🏏 CricketMatch Analytics Platform

This project builds a **real-time + batch ETL pipeline** for cricket data using **PySpark, Apache Kafka, and MySQL**.  

It ingests **historical match data** (batch) and **live ball-by-ball feeds** (streaming), validates them, transforms into **fact/dimension models**, and stores in a verified warehouse for analytics and business queries.  

---

## 📂 Project Structure

```
cricketmatch/
└── app/
    ├── config/                 # MySQL connection config
    │   └── db_config.py
    ├── etl/                    # ETL modules
    │   ├── extract.py
    │   ├── validate.py
    │   ├── transform.py
    │   └── load.py
    ├── jobs/                   # Batch + query jobs
    │   ├── etl_job.py
    │   ├── run_queries.py
    │   └── streaming_queries.py
    └── streaming/              # Kafka producer + consumer
        ├── kafka_producer.py
        └── kafka_consumer_streaming.py
```

---

## ⚙️ Setup

### 1. Python Environment
```bash
python -m venv venv
source venv/bin/activate   # (Linux/Mac)
venv\Scripts\activate      # (Windows)

pip install -r requirements.txt
```

`requirements.txt` should have:
```
pyspark
mysql-connector-python
kafka-python
```

### 2. MySQL
- Create two databases:
  ```sql
  CREATE DATABASE cricket;
  CREATE DATABASE cricket_verified;
  ```
- Place the schema/tables in `cricket` with historical + sample data.  
- Update `app/config/db_config.py` with your MySQL details:
  ```python
  DB_CONFIG = {
      "source_url": "jdbc:mysql://localhost:3306/cricket",
      "target_url": "jdbc:mysql://localhost:3306/cricket_verified",
      "user": "root",
      "password": "root",
      "driver": "com.mysql.cj.jdbc.Driver",
      "jdbc_jar": "/spark/jars/mysql-connector-j-9.4.0.jar"
  }
  ```

### 3. Kafka
Start services:
```bash
bin/zookeeper-server-start.sh config/zookeeper.properties
bin/kafka-server-start.sh config/server.properties
```

Create topic:
```bash
bin/kafka-topics.sh --create \
    --bootstrap-server localhost:9092 \
    --replication-factor 1 \
    --partitions 1 \
    --topic cricket.match.ball
```

---

## 🚀 Running the Pipeline

### 1️⃣ Batch ETL (Historical Data)
Run once to load historical records from `cricket` → `cricket_verified`:
```bash
python -m app.jobs.etl_job
```

### 2️⃣ Kafka Producer (Live Simulation)
Reads rows from `sample_t20_match` table and publishes to Kafka:
```bash
python -m app.streaming.kafka_producer
```

### 3️⃣ Kafka Consumer (Streaming ETL)
Consumes Kafka events → validates → transforms → writes into MySQL (`cricket_verified`) and registers live views in Spark:
```bash
spark-submit \
  --jars /spark/jars/mysql-connector-j-9.4.0.jar \
  app/streaming/kafka_consumer_streaming.py
```

### 4️⃣ Business Queries
- **Historical Queries** (from MySQL):
  ```bash
  python -m app.jobs.run_queries
  ```
- **Streaming Queries** (from Spark memory tables):
  ```bash
  python -m app.jobs.streaming_queries
  ```

---

## 📊 Example Queries

### Top Scorers (Historical)
```sql
SELECT playerId, SUM(runsScored) as totalRuns
FROM fact_player_match_stats
GROUP BY playerId
ORDER BY totalRuns DESC
LIMIT 10;
```

### Top Scorers (Streaming / live view)
```sql
SELECT playerId, SUM(runsScored) as totalRuns
FROM live_fact_player_match_stats
GROUP BY playerId
ORDER BY totalRuns DESC
LIMIT 10;
```

---

## 🔄 End-to-End Flow

1. **Batch ETL** loads historical data from `cricket` into `cricket_verified`.  
2. **Producer** publishes new ball events from `sample_t20_match` into Kafka topic `cricket.match.ball`.  
3. **Consumer (Spark Structured Streaming)**:
   - Validates and transforms ball events.  
   - Stores data into `fact_ballbyball` and `fact_player_match_stats` tables in `cricket_verified`.  
   - Exposes live views (`live_fact_ballbyball`, `live_fact_player_match_stats`) for in-memory queries.  
4. **Queries** run either on historical warehouse (MySQL) or on live Spark views (streaming).  

---

## 🛠️ Notes

- Always run from project root (`cricketmatch/`).  
- Use `python -m app.jobs...` or `spark-submit app/streaming/...` so that Python packages resolve correctly.  
- If `mysql-connector` errors → ensure `mysql-connector-python` is installed for Python and JAR exists for Spark.  
- Kafka + Zookeeper must be running before producer/consumer.  
