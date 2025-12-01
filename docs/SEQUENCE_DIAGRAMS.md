# Sequence Diagrams - Real-Time Analytics Platform

## 1. Data Ingestion Flow (Luồng Nhập Dữ Liệu)

Sơ đồ này mô tả luồng dữ liệu từ khi Airflow sinh message cho đến khi dữ liệu được lưu trong Druid.

```plantuml
@startuml
title Data Ingestion Flow - Real-Time Analytics

actor User
participant "Airflow\nScheduler" as Scheduler
participant "Demo DAG\n(demo.py)" as DAG
participant "Kafka Producer" as Producer
participant "Kafka Broker\n(topic: demo)" as Kafka
participant "Druid\nMiddleManager" as MiddleManager
participant "Druid\nCoordinator" as Coordinator
participant "PostgreSQL\n(Metadata)" as Postgres
participant "Druid\nHistorical" as Historical
database "Local Storage\n(/opt/shared)" as Storage

== Initialization Phase ==
User -> Scheduler: Start Airflow
activate Scheduler
Scheduler -> DAG: Load DAG (schedule: */1 * * * *)
activate DAG
DAG --> Scheduler: DAG registered
deactivate DAG

MiddleManager -> Kafka: Subscribe to topic "demo"
activate MiddleManager
activate Kafka

== Every Minute Execution ==
Scheduler -> DAG: Trigger DAG run
activate DAG

loop For each cryptocurrency (BTC, ETH, DOT, BTT)
    DAG -> DAG: Generate message\n{data_id, name, timestamp}
    DAG -> Producer: Send message
    activate Producer
    Producer -> Kafka: Produce to topic "demo"
    Kafka --> Producer: ACK
    deactivate Producer
    note right of Kafka
        Message stored in partition
        with configurable retention
    end note
end

DAG --> Scheduler: Task completed
deactivate DAG

== Real-Time Ingestion ==
Kafka -> MiddleManager: Poll messages
activate MiddleManager

MiddleManager -> MiddleManager: Parse & validate JSON
MiddleManager -> MiddleManager: Create real-time segment\n(in-memory indexing)

note right of MiddleManager
    Segment buffering:
    - Time-based chunking
    - Columnar format
    - Compressed storage
end note

MiddleManager -> Coordinator: Register segment metadata
activate Coordinator
Coordinator -> Postgres: Store segment info
activate Postgres
Postgres --> Coordinator: Metadata saved
deactivate Postgres
Coordinator --> MiddleManager: Segment registered
deactivate Coordinator

== Segment Handoff ==
MiddleManager -> Storage: Persist segment to disk\n(/opt/shared/segments/)
activate Storage
Storage --> MiddleManager: Segment persisted
deactivate Storage

MiddleManager -> Coordinator: Notify segment available
activate Coordinator
Coordinator -> Historical: Assign segment
activate Historical

Historical -> Storage: Load segment from deep storage
activate Storage
Storage --> Historical: Segment loaded
deactivate Storage

Historical -> Postgres: Update segment status
activate Postgres
Postgres --> Historical: Status updated
deactivate Postgres

Historical --> Coordinator: Segment ready
deactivate Historical
Coordinator --> MiddleManager: Handoff completed
deactivate Coordinator

MiddleManager -> MiddleManager: Drop real-time segment
deactivate MiddleManager

note over Historical, Storage
    Historical node now serves
    this segment for queries
end note

@enduml
```

---

## 2. Query Flow (Luồng Truy Vấn)

Sơ đồ này mô tả cách một query SQL từ Superset được xử lý qua các components của Druid.

```plantuml
@startuml
title Query Execution Flow - Apache Druid

actor User
participant "Superset\nDashboard" as Superset
participant "Druid Router\n(:8888)" as Router
participant "Druid Broker\n(:8082)" as Broker
participant "Druid Historical\n(:8083)" as Historical
participant "Druid\nMiddleManager\n(:8091)" as MiddleManager
participant "PostgreSQL\n(Metadata)" as Postgres
database "Local Storage" as Storage

== User Query Initiation ==
User -> Superset: View dashboard /\nExecute SQL query
activate Superset

Superset -> Superset: Generate Druid SQL:\nSELECT name, AVG(data_id)\nFROM demo\nWHERE __time > NOW() - INTERVAL '1' HOUR\nGROUP BY name

Superset -> Router: POST /druid/v2/sql/\n(SQLAlchemy connection)
activate Router

Router -> Router: Parse request\nDetermine query type
Router -> Broker: Forward SQL query
activate Broker

== Query Planning ==
Broker -> Postgres: Fetch segment metadata\nfor datasource "demo"
activate Postgres
Postgres --> Broker: Return segment list\n+ time ranges + locations
deactivate Postgres

Broker -> Broker: Query Planning:\n1. Parse SQL to native query\n2. Prune segments by time range\n3. Identify target nodes

note right of Broker
    Query Optimization:
    - Time-based pruning
    - Filter pushdown
    - Aggregation planning
end note

== Parallel Query Execution ==
par Historical Segments
    Broker -> Historical: Query historical segments
    activate Historical

    Historical -> Storage: Read segment data
    activate Storage
    Storage --> Historical: Return compressed columns
    deactivate Storage

    Historical -> Historical: Scan & filter rows\nApply aggregations
    Historical --> Broker: Partial results (aggregated)
    deactivate Historical
else Real-time Segments
    Broker -> MiddleManager: Query real-time segments
    activate MiddleManager

    MiddleManager -> MiddleManager: Scan in-memory segments\nApply filters & aggregations
    MiddleManager --> Broker: Partial results (aggregated)
    deactivate MiddleManager
end

== Result Merging ==
Broker -> Broker: Merge partial results:\n1. Combine aggregations\n2. Apply LIMIT/ORDER BY\n3. Format response

note right of Broker
    Result Merge:
    - TopN merge
    - GroupBy merge
    - Timeseries merge
end note

Broker --> Router: Return merged results (JSON)
deactivate Broker

Router --> Superset: HTTP 200 + JSON payload
deactivate Router

== Visualization ==
Superset -> Superset: Parse JSON results
Superset -> Superset: Render chart/table
Superset --> User: Display dashboard
deactivate Superset

note over User, Superset
    Typical query latency:
    - Simple aggregations: <100ms
    - Complex queries: 100ms-2s
    - Depends on data volume
end note

@enduml
```

---

## 3. System Startup Sequence (Khởi Động Hệ Thống)

Sơ đồ này mô tả thứ tự khởi động và dependencies giữa các services khi chạy `docker-compose up`.

```plantuml
@startuml
title System Startup Sequence - Docker Compose

participant "Docker\nCompose" as Compose
participant "Zookeeper\n(:2181)" as Zookeeper
participant "PostgreSQL\n(:5432)" as Postgres
participant "Kafka\n(:9092)" as Kafka
participant "Druid\nCoordinator" as Coordinator
participant "Druid\nBroker" as Broker
participant "Druid\nHistorical" as Historical
participant "Druid\nMiddleManager" as MiddleManager
participant "Druid\nRouter" as Router
participant "Airflow\n(:3000)" as Airflow
participant "Superset\n(:8088)" as Superset

== Infrastructure Layer ==
Compose -> Zookeeper: Start container
activate Zookeeper
Zookeeper -> Zookeeper: Initialize ZK data dir\nStart ZK server
note right: Port 2181 ready

Compose -> Postgres: Start container
activate Postgres
Postgres -> Postgres: Initialize database\nCreate druid metadata DB
note right: Port 5432 ready

== Streaming Layer ==
Compose -> Kafka: Start container (depends_on: zookeeper)
activate Kafka

Kafka -> Zookeeper: Connect for cluster coordination
Zookeeper --> Kafka: Connection established

Kafka -> Kafka: Initialize broker\nCreate topic "demo" (auto-create enabled)
note right: Port 9092 ready

== Druid Cluster - Master Services ==
Compose -> Coordinator: Start container (depends_on: postgres, zookeeper)
activate Coordinator

Coordinator -> Postgres: Connect to metadata DB
Postgres --> Coordinator: Connection OK

Coordinator -> Zookeeper: Register in cluster
Zookeeper --> Coordinator: Registered

Coordinator -> Coordinator: Initialize:\n- Load extensions\n- Start HTTP server\n- Load rules & configs
note right: Port 8081 ready

Compose -> Broker: Start container (depends_on: postgres, zookeeper)
activate Broker

Broker -> Postgres: Connect to metadata DB
Postgres --> Broker: Connection OK

Broker -> Zookeeper: Register in cluster
Zookeeper --> Broker: Registered

Broker -> Broker: Initialize query engine
note right: Port 8082 ready

== Druid Cluster - Data Services ==
Compose -> Historical: Start container (depends_on: postgres, zookeeper)
activate Historical

Historical -> Postgres: Connect to metadata DB
Postgres --> Historical: Connection OK

Historical -> Zookeeper: Register in cluster
Zookeeper --> Historical: Registered

Historical -> Coordinator: Announce available capacity
Coordinator --> Historical: ACK

note right of Historical: Port 8083 ready

Compose -> MiddleManager: Start container (depends_on: postgres, zookeeper)
activate MiddleManager

MiddleManager -> Postgres: Connect to metadata DB
Postgres --> MiddleManager: Connection OK

MiddleManager -> Zookeeper: Register in cluster
Zookeeper --> MiddleManager: Registered

MiddleManager -> Coordinator: Announce available capacity
Coordinator --> MiddleManager: ACK

note right of MiddleManager: Ports 8091, 8100-8105 ready

== Druid Router ==
Compose -> Router: Start container (depends_on: postgres, zookeeper)
activate Router

Router -> Postgres: Connect to metadata DB
Postgres --> Router: Connection OK

Router -> Zookeeper: Discover cluster topology
Zookeeper --> Router: Return broker/coordinator locations

Router -> Router: Initialize routing table
note right: Port 8888 ready (unified endpoint)

== Kafka Ingestion Setup ==
MiddleManager -> Kafka: Subscribe to topic "demo"
Kafka --> MiddleManager: Subscription confirmed

note over MiddleManager, Kafka
    MiddleManager starts polling
    for messages on topic "demo"
end note

== Orchestration Layer ==
Compose -> Airflow: Start container (depends_on: postgres)
activate Airflow

Airflow -> Airflow: Initialize:\n- Create SQLite DB (airflow.db)\n- Load DAGs from /airflow/dags\n- Start scheduler\n- Start webserver

Airflow -> Airflow: Register DAG "Demo"\nSchedule: */1 * * * *

note right: Port 3000 ready

== Visualization Layer ==
Compose -> Superset: Start container
activate Superset

Superset -> Superset: Initialize:\n- Create admin user\n- Load example dashboards\n- Start web server

note right: Port 8088 ready

== Health Check ==
Router -> Broker: Health check
Broker --> Router: OK

Router -> Coordinator: Health check
Coordinator --> Router: OK

Airflow -> Kafka: Test connection
Kafka --> Airflow: Connection OK

Superset -> Router: Test Druid connection\n(druid://broker:8082/druid/v2/sql/)
Router --> Superset: Connection OK

note over Compose
    All services ready!
    System is operational
end note

@enduml
```

---

## 4. Airflow DAG Execution Detail

Sơ đồ chi tiết về cách DAG "Demo" thực thi mỗi phút.

```plantuml
@startuml
title Airflow DAG Execution - Demo Producer

participant "Airflow\nScheduler" as Scheduler
participant "DAG: Demo\n(demo.py)" as DAG
participant "Task:\ndemo_func()" as Task
participant "KafkaProducer\nClient" as Producer
participant "Kafka Broker" as Kafka

== Every Minute (Cron: */1 * * * *) ==
Scheduler -> Scheduler: Check schedule\nTime: XX:XX:00

Scheduler -> DAG: Create DagRun instance
activate DAG

DAG -> Task: Execute PythonOperator
activate Task

Task -> Task: Import kafka-python library

Task -> Producer: Initialize KafkaProducer\nbootstrap_servers='kafka:9092'
activate Producer
Producer -> Kafka: Establish TCP connection
activate Kafka
Kafka --> Producer: Connection established

loop For each coin in ["BTC", "ETH", "DOT", "BTT"]
    Task -> Task: Generate random data_id:\n- BTC: random(100, 200)\n- ETH: random(80, 150)\n- DOT: random(20, 50)\n- BTT: random(1, 10)

    Task -> Task: Get current timestamp:\ntimestamp = int(time.time())

    Task -> Task: Create message:\n{\n  "data_id": random_value,\n  "name": coin,\n  "timestamp": timestamp\n}

    Task -> Producer: send(\n  topic="demo",\n  value=json.dumps(message)\n)

    Producer -> Producer: Serialize message to bytes
    Producer -> Kafka: Send to partition (hash by key)
    Kafka -> Kafka: Append to commit log
    Kafka --> Producer: Offset + partition info
    Producer --> Task: Send success

    note right of Kafka
        Message persisted:
        - Topic: demo
        - Partition: 0 (single partition)
        - Offset: auto-increment
    end note
end

Task -> Producer: flush()
Producer -> Kafka: Flush buffer
Kafka --> Producer: All messages committed

Task -> Producer: close()
deactivate Producer

Task --> DAG: Task success
deactivate Task

DAG -> DAG: Mark DagRun as success
DAG --> Scheduler: DagRun completed
deactivate DAG

Scheduler -> Scheduler: Update task status in DB
Scheduler -> Scheduler: Schedule next run: XX:XX+1:00

note over Scheduler, Kafka
    Total messages per minute: 4
    Total messages per hour: 240
    Total messages per day: 5,760
end note

@enduml
```

---

## 5. Superset Dashboard Load Sequence

Sơ đồ mô tả cách Superset load dashboard và query Druid.

```plantuml
@startuml
title Superset Dashboard Rendering

actor User
participant "Browser" as Browser
participant "Superset\nWebserver" as Superset
participant "Superset\nMetadata DB" as SupersetDB
participant "Druid Router" as Router
participant "Druid Broker" as Broker

== Dashboard Access ==
User -> Browser: Navigate to\nhttp://localhost:8088/dashboard/1
Browser -> Superset: GET /dashboard/1
activate Superset

Superset -> SupersetDB: SELECT * FROM dashboards WHERE id=1
activate SupersetDB
SupersetDB --> Superset: Dashboard metadata:\n- Title, layout, charts[]
deactivate SupersetDB

Superset --> Browser: HTML + JavaScript
deactivate Superset
Browser -> Browser: Render dashboard layout

== Chart Data Loading ==
loop For each chart in dashboard
    Browser -> Superset: POST /superset/explore_json/\n{\n  datasource: "demo",\n  metrics: ["AVG(data_id)"],\n  groupby: ["name"],\n  time_range: "Last 1 hour"\n}
    activate Superset

    Superset -> Superset: Build SQL query:\nSELECT name,\n       AVG(data_id) as avg_value\nFROM demo\nWHERE __time >= CURRENT_TIMESTAMP - INTERVAL '1' HOUR\nGROUP BY name

    Superset -> Router: POST /druid/v2/sql/\nContent-Type: application/json\n{\n  "query": "SELECT ...",\n  "context": {...}\n}
    activate Router

    Router -> Broker: Forward SQL query
    activate Broker

    Broker -> Broker: Execute query\n(see Query Flow diagram)

    Broker --> Router: Results:\n[\n  {"name":"BTC", "avg_value":150.5},\n  {"name":"ETH", "avg_value":115.2},\n  ...\n]
    deactivate Broker

    Router --> Superset: HTTP 200 + JSON
    deactivate Router

    Superset -> Superset: Format data for chart type\n(timeseries, bar, pie, etc.)

    Superset --> Browser: JSON response
    deactivate Superset

    Browser -> Browser: Render chart using\nReact + D3.js / ECharts
end

Browser --> User: Display complete dashboard

note over User, Browser
    Dashboard auto-refresh:
    Can be configured to
    refresh every N seconds
end note

@enduml
```

---

## Giải Thích Các Flow

### 1. Data Ingestion Flow
**Mục đích:** Đưa dữ liệu từ source vào Druid để có thể query được

**Các bước chính:**
1. **Scheduling (0-1s):** Airflow scheduler trigger DAG mỗi phút
2. **Production (1-2s):** DAG sinh 4 messages và gửi vào Kafka topic "demo"
3. **Buffering (2-3s):** Kafka lưu messages vào partition với durability
4. **Real-time Ingestion (3-5s):** MiddleManager consume messages và tạo real-time segments in-memory
5. **Indexing (real-time):** Data được index dạng columnar, compressed
6. **Handoff (mỗi 10 phút):** Real-time segments được persist xuống disk và hand-off cho Historical nodes
7. **Long-term Storage:** Historical nodes serve data cho queries

**Latency:** Sub-second từ Kafka đến queryable trong Druid

---

### 2. Query Flow
**Mục đích:** Thực thi SQL query và trả về kết quả

**Các bước chính:**
1. **User Input:** User viết SQL trong Superset hoặc gọi API
2. **Routing (Router):** Router nhận request và forward đến Broker
3. **Planning (Broker):**
   - Parse SQL thành native Druid query
   - Fetch segment metadata từ PostgreSQL
   - Prune segments không liên quan (time range filtering)
   - Xác định Historical/MiddleManager nodes cần query
4. **Parallel Execution:**
   - Query Historical nodes cho historical segments
   - Query MiddleManager cho real-time segments
   - Mỗi node thực thi partial aggregation
5. **Merge (Broker):** Merge kết quả từ tất cả nodes
6. **Return:** Trả JSON về cho client

**Performance:**
- Simple queries: <100ms
- Complex aggregations: 100ms-2s
- Scan queries: tùy data volume

---

### 3. System Startup
**Mục đích:** Khởi động toàn bộ stack theo đúng dependencies

**Thứ tự:**
1. **Infrastructure:** Zookeeper, PostgreSQL (không phụ thuộc gì)
2. **Streaming:** Kafka (phụ thuộc Zookeeper)
3. **Druid Masters:** Coordinator, Broker (phụ thuộc Postgres + Zookeeper)
4. **Druid Data:** Historical, MiddleManager (phụ thuộc Postgres + Zookeeper)
5. **Druid Router:** Router (phụ thuộc Postgres + Zookeeper + Broker + Coordinator)
6. **Apps:** Airflow, Superset (phụ thuộc các services đã ready)

**Health checks:** Mỗi service tự kiểm tra dependencies trước khi declare "ready"

---

### 4. DAG Execution
**Mục đích:** Tự động sinh dữ liệu demo mỗi phút

**Chi tiết:**
- **Trigger:** Cron schedule `*/1 * * * *`
- **Execution time:** ~2-3 seconds
- **Messages per run:** 4 (BTC, ETH, DOT, BTT)
- **Data format:** JSON với random data_id và real timestamp
- **Delivery:** At-least-once delivery guarantee từ Kafka

---

### 5. Dashboard Rendering
**Mục đích:** Hiển thị real-time analytics cho user

**Workflow:**
1. User truy cập dashboard URL
2. Superset load dashboard config từ metadata DB
3. Superset render HTML + charts layout
4. Browser gọi API cho từng chart để lấy data
5. Superset chuyển đổi chart config thành Druid SQL
6. Druid thực thi query và trả về data
7. Browser render charts bằng JavaScript libraries

**Caching:** Superset có thể cache query results để giảm load lên Druid

---

## Notes về PlantUML

Để render các diagrams trên, bạn có thể:

1. **VS Code:** Cài extension "PlantUML"
2. **Online:** https://www.plantuml.com/plantuml/uml/
3. **CLI:** `plantuml diagram.puml` (cần Java + Graphviz)
4. **IntelliJ IDEA:** Built-in PlantUML support

Hoặc sử dụng Markdown viewers hỗ trợ PlantUML như GitLab, GitHub (với plugins), hoặc Obsidian.
