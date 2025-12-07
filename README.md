# City-Scale Real-Time Transit Tracking & ETA System

## 📖 Executive Summary

[cite_start]This project implements a high-throughput, Event-Driven Architecture (EDA) designed to track over 20,000 transit vehicles in real-time[cite: 6]. [cite_start]The system ingests high-frequency telemetry, resolves geospatial ambiguities via map matching, calculates dynamic ETAs using machine learning, and disseminates standard GTFS-Realtime feeds with sub-second latency[cite: 6].

[cite_start]The architecture is designed to handle "Thundering Herd" scenarios and solves the "ghost bus" phenomenon by ensuring data freshness and strictly controlled end-to-end latency (\< 3 seconds)[cite: 13, 16, 39].

-----

## 🏗 System Architecture

[cite_start]The system follows **Event Sourcing** and **CQRS** patterns to decouple high-velocity write workloads from high-variability read patterns[cite: 60, 61].

### The 5-Layer Stack

1.  [cite_start]**Ingestion Layer (Edge):** Stateless Go microservices handling MQTT 5.0 connectivity and mTLS authentication[cite: 64, 86].
2.  [cite_start]**Streaming Backbone:** Apache Kafka acting as the single source of truth for all events[cite: 65].
3.  [cite_start]**Processing Layer:** Apache Flink for stateful stream processing (Map Matching & ETA calculation)[cite: 67].
4.  [cite_start]**Persistence Layer:** Polyglot storage using Redis (Geospatial), ScyllaDB (History), and PostGIS (Static Schedule)[cite: 68].
5.  [cite_start]**Service Layer:** API Gateway exposing GTFS-Realtime feeds and REST endpoints[cite: 72].

### Data Flow Pipeline

[cite_start]The data flow is unidirectional and reactive[cite: 74]:

1.  [cite_start]**Vehicle:** Sends telemetry (MQTT/TLS) $\rightarrow$ **Gateway**[cite: 75].
2.  [cite_start]**Gateway:** Validates & Publishes to `telemetry.raw` $\rightarrow$ **Kafka**[cite: 76].
3.  [cite_start]**Flink (Job 1):** Consumes raw data, snaps to road graph $\rightarrow$ Publishes to `telemetry.matched`[cite: 77].
4.  [cite_start]**Flink (Job 2):** Consumes matched data, calculates ETA $\rightarrow$ Publishes to `trip.updates`[cite: 78].
5.  [cite_start]**Sinks:** Writes final state to **Redis** (Live) and **ScyllaDB** (History)[cite: 79].
6.  [cite_start]**API:** Fetches from Redis $\rightarrow$ Returns response to **User**[cite: 80].

-----

## 🛠 Technology Stack

| Component | Technology | Rationale |
| :--- | :--- | :--- |
| **Language** | **Go (Golang)** | [cite_start]High concurrency primitives and low memory footprint for handling 50k+ connections[cite: 82]. |
| **Stream Processing** | **Apache Flink** | [cite_start]Superior state management and "exactly-once" semantics compared to Spark Streaming[cite: 82]. |
| **Message Bus** | **Apache Kafka** | [cite_start]Ability to handle massive throughput (50k EPS) with low latency (5ms at p99)[cite: 45, 82]. |
| **Hot Storage** | **Redis (Geospatial)** | [cite_start]O(log(N)) geospatial indexing for sub-millisecond "buses near me" queries[cite: 82]. |
| **Warm Storage** | **ScyllaDB** | [cite_start]High-throughput writes without JVM garbage collection pauses; optimized for time-series[cite: 82]. |
| **Static Storage** | **PostgreSQL + PostGIS** | [cite_start]Complex spatial joins for static schedule and topology management[cite: 71, 82]. |

-----

## ⚡ Key Features

### 1\. Robust Telemetry Ingestion

  * [cite_start]**Protocol:** Uses MQTT 5.0 for bandwidth efficiency (saving GBs daily) and "Last Will and Testament" for instant disconnection detection[cite: 86, 88, 89].
  * [cite_start]**Normalization:** Converts NMEA 0183 sentences, JSON, or Protobuf into a canonical internal schema[cite: 21].
  * [cite_start]**Throughput:** Supports sustained ingestion of 50,000 events per second (EPS)[cite: 45].

### 2\. Advanced Map Matching

  * [cite_start]Utilizes Hidden Markov Models (HMM) or geometric projection to snap raw, noisy GPS points to the GTFS static road network[cite: 116].
  * [cite_start]Filters out topologically impossible jumps (e.g., bridge vs. river road)[cite: 117].

### 3\. Tiered ETA Calculation

[cite_start]The system uses a hybrid prediction engine[cite: 155]:

  * [cite_start]**Tier 1 (Baseline):** Schedule Deviation ($T_{actual} - T_{scheduled}$)[cite: 156].
  * [cite_start]**Tier 2 (Historical):** Segment-based profiling bucketed by day/time[cite: 161].
  * [cite_start]**Tier 3 (ML - Advanced):** XGBoost/LSTM models considering weather, real-time traffic, and headway[cite: 167, 168].

### 4\. Security & Compliance

  * [cite_start]**Zero Trust:** All vehicle connections authenticated via Mutual TLS (mTLS)[cite: 55].
  * [cite_start]**GPS Spoofing Detection:** Physics-based anomaly detection rejects data indicating impossible speeds (\>120 km/h) or teleportation[cite: 190, 191].
  * [cite_start]**Standard Output:** Fully compatible with Google Transit (GTFS-Realtime) specification via Protocol Buffers[cite: 31].

-----

## 💾 Data Schema & Storage

### Historical Telemetry (ScyllaDB)

[cite_start]Optimized for "Query by Vehicle by Time" using TimeWindowCompactionStrategy[cite: 142, 145].

```sql
CREATE TABLE telemetry_history (
    vehicle_id text,
    bucket_day text,
    timestamp timestamp,
    lat double,
    lon double,
    speed double,
    trip_id text,
    PRIMARY KEY ((vehicle_id, bucket_day), timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC)
  AND compaction = {'class': 'TimeWindowCompactionStrategy', ...};
```

[cite_start]*[cite: 143]*

### Real-Time State (Redis)

  * [cite_start]**Geospatial Index:** `GEOADD transit:locations <lon> <lat> <vehicle_id>`[cite: 136].
  * [cite_start]**Metadata:** `HASH vehicle:{id}` stores trip IDs and occupancy status[cite: 138].

-----

## 🚀 API Reference

### GTFS-Realtime Feeds

[cite_start]Feeds are generated via background workers and served via CDN[cite: 176].

  * `GET /gtfs-rt/vehicle-positions.pb`
  * `GET /gtfs-rt/trip-updates.pb`
  * `GET /gtfs-rt/alerts.pb`

### Developer REST API

[cite_start]Rate-limited via Token Bucket algorithm (Redis backed)[cite: 184].

  * [cite_start]`GET /vehicles?route={id}`: Returns live positions for a specific route[cite: 182].
  * [cite_start]`GET /stops/{id}/predictions`: Returns upcoming arrivals[cite: 183].

-----

## 📊 Service Level Objectives (SLOs)

| Metric | Target | Description |
| :--- | :--- | :--- |
| **End-to-End Latency** | 95% \< 3s | [cite_start]Time from sensor event to API availability[cite: 58]. |
| **Ingestion Success** | 99.99% | [cite_start]Percentage of successful HTTP/MQTT acks[cite: 58]. |
| **Map Matching Accuracy**| \> 98% | [cite_start]Points successfully snapped to valid graph edge[cite: 58]. |
| **Availability** | 99.99% | [cite_start]Max \~52 minutes downtime per year[cite: 49]. |

-----

## 🛡 Operational Resilience

  * [cite_start]**Disaster Recovery:** Active-Passive deployment across two regions with Kafka MirrorMaker 2 replication[cite: 207].
  * [cite_start]**Observability:** OpenTelemetry distributed tracing from Gateway to Redis[cite: 210].
  * [cite_start]**Self-Healing:** Automatic consumer group rebalancing in Kafka and DB failover[cite: 51].

-----

## 📄 License

[Insert License Here]

-----

[cite_start]*Based on the "City-Scale Real-Time Transit Tracking & ETA System: System Design Report"[cite: 1].*
