# Water Quality Monitoring Platform - Backend Architecture

## 1. Overview
The **Water Quality Monitoring Platform** is designed to track and analyze water quality in real-time across **municipal, industrial, and natural water sources**. The platform leverages:
- **IoT sensors** for real-time data collection.
- **Data analytics & AI** for contamination detection and predictive insights.
- **Scalable backend services** to handle millions of sensors and high transaction volumes.
- **APIs** for seamless integration with external systems.

The architecture is built for **high availability, scalability, and fault tolerance**, ensuring real-time decision-making for safe and sustainable water usage.

---

## 2. High-Level Architecture
The platform's backend is designed to support **10 million IoT sensors**, with the following key components:

### **Core Components:**
- **IoT Sensors:** Collect real-time data on **pH, turbidity, dissolved oxygen, and temperature**.
- **DNS Load Balancers:** Distribute traffic to regional load balancers.
- **Load Balancer Cluster:** Routes requests to the **App Server**.
- **App Server:** Manages data ingestion, processing, and routing.
- **Core Services:**
  - **Water Quality Monitoring:** Detects contaminants and threshold breaches.
  - **Notification Service:** Sends real-time alerts.
  - **Analytics Service:** Generates predictive insights and maintenance reports.
- **Storage Layer:**
  - **PostgreSQL:** Stores structured data.
  - **AWS S3:** Stores historical data for analysis.
- **Outputs:** User dashboards for visualization and decision-making.

---

## 3. Key Backend Design Decisions

### **Task Prioritization**
To ensure critical tasks are executed first, tasks are categorized into three priority levels:

| Priority | Task Type | Processing Time |
|----------|-------------------------|----------------|
| High | Contamination Alerts | <300ms |
| Medium | Dashboard Updates | <1s |
| Low | Analytics & Reports | Batch processing |

**RabbitMQ** handles prioritization using **priority queues**:
- `high_priority_queue` → Contamination alerts
- `medium_priority_queue` → Dashboard updates
- `low_priority_queue` → Reports and analytics

---

### **Message Queue Architecture**

#### **Why Message Queues?**
- **Decouples ingestion from processing**, ensuring scalability.
- **Prevents bottlenecks** by buffering high-throughput data streams.
- **Ensures fault tolerance** in case of failures.

#### **Queue Implementation**
- **RabbitMQ (for real-time processing)**:
  - Direct exchange routing with priority-based queues.
  - Workers consume messages based on priority levels.
- **Kafka (for high-volume sensor ingestion)**:
  - Partitions data by **region** or **sensor type**.
  - Batch consumers process data asynchronously.

##### **Queue Throughput Calculation**
- **Total tasks per second:** 1,000,000
- **High priority (50%)**: 500,000 tasks/sec
- **Medium priority (30%)**: 300,000 tasks/sec
- **Low priority (20%)**: 200,000 tasks/sec
- **RabbitMQ Node Throughput**: 50,000 messages/sec
- **Required RabbitMQ nodes**:
  ```
  Nodes = Total throughput / Throughput per node
  Nodes = 500,000 / 50,000 = 10 (for high-priority tasks)
  ```

---

## 4. Database Schema & Scalability

### **Database Selection**
- **PostgreSQL** for structured, real-time sensor data.
- **AWS S3** for long-term storage and compliance.

### **PostgreSQL Schema (Sharded for Scalability)**
```sql
CREATE TABLE sensor_readings (
    sensor_id BIGINT NOT NULL,
    timestamp TIMESTAMP NOT NULL,
    ph_level FLOAT,
    turbidity FLOAT,
    temperature FLOAT,
    dissolved_oxygen FLOAT,
    region VARCHAR(20),
    PRIMARY KEY (sensor_id, timestamp)
) PARTITION BY HASH (sensor_id);
```

```sql
CREATE TABLE contamination_alerts (
    alert_id BIGSERIAL PRIMARY KEY,
    sensor_id BIGINT NOT NULL,
    timestamp TIMESTAMP NOT NULL,
    contaminant_type VARCHAR(50),
    severity_level INT,
    resolved BOOLEAN DEFAULT FALSE
);
```

### **AWS S3 Storage for Historical Data**
- Bucket structure:
  ```plaintext
  s3://water_quality_data/<region>/<year>/<month>/<day>/
  ```
- Example File:
  ```json
  {
      "sensor_id": 123,
      "readings": [
          { "timestamp": "2025-01-27T12:00:00Z", "ph_level": 7.2, "turbidity": 1.5, "temperature": 21.0 }
      ]
  }
  ```

---

### **Sharding & Scaling Calculations**

#### **Sharding Strategy**
- **10,000,000 sensors**, partitioned across **32 shards**.
- **Sharding Key:** `sensor_id` (hash-based partitioning).

#### **Transaction Throughput Per Shard**
1. **Sensors per shard**:
   ```
   Sensors per shard = Total sensors / Shards
   = 10,000,000 / 32 = 312,500
   ```
2. **TPS per sensor**:
   ```
   Each sensor sends 1 record every 10 seconds → 0.1 TPS per sensor.
   ```
3. **Total TPS per shard**:
   ```
   TPS per shard = 312,500 * 0.1 = 31,250 TPS
   ```
4. **PostgreSQL Limit**:
   - PostgreSQL supports **30,000 TPS per table**.
   - **Solution**: Increase shards from **32 → 40**, reducing TPS per shard to **25,000 TPS**, ensuring scalability.

---

## 5. Future Scalability & Optimizations

### **Immediate Scaling Solutions**
- **Increase PostgreSQL shards from 32 → 40** for better TPS distribution.
- **Add more RabbitMQ nodes** for task queue scaling.

### **Long-Term Enhancements**
1. **Edge Processing**:
   - Deploy edge servers for preprocessing sensor data before cloud ingestion.
2. **API Expansion**:
   - Add REST & GraphQL APIs for new industry partners.
3. **Advanced Analytics**:
   - Integrate **machine learning models** for anomaly detection & forecasting.

---

## 6. Conclusion
The **backend architecture** for the Water Quality Monitoring Platform is designed for **high throughput, fault tolerance, and scalability**. By implementing **task prioritization, message queues, database sharding, and caching strategies**, the system can handle **millions of sensors while maintaining low-latency responses**.

---
