# Chi Tiết Các Layer và Tech Stack

## Mục Lục
1. [Tổng Quan Kiến Trúc](#tổng-quan-kiến-trúc)
2. [Layer 1: Orchestration & Workflow](#layer-1-orchestration--workflow)
3. [Layer 2: Data Streaming](#layer-2-data-streaming)
4. [Layer 3: Analytics Engine](#layer-3-analytics-engine)
5. [Layer 4: Storage](#layer-4-storage)
6. [Layer 5: Visualization](#layer-5-visualization)
7. [Infrastructure & DevOps](#infrastructure--devops)
8. [Data Flow Chi Tiết](#data-flow-chi-tiết)
9. [Production Best Practices](#production-best-practices)

---

## Tổng Quan Kiến Trúc

Hệ thống sử dụng **Lambda Architecture** kết hợp với **Event-Driven Architecture**:

```
┌─────────────────────────────────────────────────────────────┐
│                    LAMBDA ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐         ┌──────────────┐                 │
│  │  Speed Layer │         │  Batch Layer │                 │
│  │   (Druid     │         │   (Future:   │                 │
│  │ Real-time    │         │    Spark)    │                 │
│  │  Ingestion)  │         │              │                 │
│  └──────────────┘         └──────────────┘                 │
│         │                        │                          │
│         └────────────┬───────────┘                          │
│                      ▼                                       │
│              ┌──────────────┐                               │
│              │ Serving Layer│                               │
│              │    (Druid    │                               │
│              │    Query)    │                               │
│              └──────────────┘                               │
└─────────────────────────────────────────────────────────────┘
```

**Đặc điểm:**
- ✅ Real-time ingestion với latency < 1s
- ✅ OLAP queries với sub-second response time
- ✅ Horizontal scalability
- ✅ Fault-tolerant với message replay từ Kafka
- ✅ Columnar storage với high compression ratio

---

## Layer 1: Orchestration & Workflow

### Apache Airflow 2.2.5

#### Vai Trò
Airflow là "conductor" của toàn bộ data pipeline, chịu trách nhiệm:
- ⏰ **Scheduling:** Trigger tasks theo schedule (cron expressions)
- 🔄 **Workflow Management:** Quản lý dependencies giữa các tasks
- 📊 **Monitoring:** Track task status, retry failed tasks
- 🔧 **Orchestration:** Coordinate giữa nhiều systems

#### Components

##### 1. Airflow Scheduler
```python
# Pseudo-code của scheduler loop
while True:
    dags = load_dags_from("/airflow/dags")
    for dag in dags:
        if should_trigger(dag.schedule_interval):
            create_dag_run(dag)

    for task in get_scheduled_tasks():
        if dependencies_met(task):
            execute_task(task)

    sleep(scheduler_heartbeat)
```

**Configuration:**
- `executor`: SequentialExecutor (demo) / CeleryExecutor (production)
- `parallelism`: 32 tasks globally
- `dag_concurrency`: 16 tasks per DAG
- `scheduler_heartbeat_sec`: 5 seconds

##### 2. Airflow Webserver
- **Port:** 3000 (external), 8080 (internal)
- **Framework:** Flask + WTForms
- **Authentication:** Basic auth (demo) / LDAP, OAuth (production)
- **Features:**
  - DAG visualization (Graph View, Tree View, Gantt)
  - Task logs viewer
  - Variable & connection management
  - Manual trigger & backfill

##### 3. Metadata Database
- **Type:** SQLite (demo) → PostgreSQL/MySQL (production)
- **Schema:**
  - `dag`: DAG definitions
  - `dag_run`: DAG execution instances
  - `task_instance`: Task execution records
  - `xcom`: Cross-communication data
  - `variable`: Airflow variables
  - `connection`: External system credentials

#### DAG Configuration (demo.py)

```python
# File: /app_airflow/app/dags/demo.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'airflow',
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG(
    dag_id='Demo',
    default_args=default_args,
    description='Produce crypto price data to Kafka',
    schedule_interval='*/1 * * * *',  # Every minute
    start_date=datetime(2023, 1, 1),
    catchup=False,  # Don't backfill
)

def demo_func():
    """Generate and send messages to Kafka"""
    from kafka import KafkaProducer
    import json, random, time

    producer = KafkaProducer(
        bootstrap_servers=['kafka:9092'],
        value_serializer=lambda v: json.dumps(v).encode('utf-8')
    )

    coins = {
        'BTC': (100, 200),
        'ETH': (80, 150),
        'DOT': (20, 50),
        'BTT': (1, 10),
    }

    for name, (min_val, max_val) in coins.items():
        message = {
            'data_id': random.randint(min_val, max_val),
            'name': name,
            'timestamp': int(time.time()),
        }
        producer.send('demo', value=message)

    producer.flush()
    producer.close()

task = PythonOperator(
    task_id='produce_crypto_data',
    python_callable=demo_func,
    dag=dag,
)
```

#### Tech Stack
| Component | Technology | Version | Purpose |
|-----------|-----------|---------|---------|
| Core | Apache Airflow | 2.2.5 | Workflow engine |
| Language | Python | 3.9 | DAG development |
| Executor | SequentialExecutor | - | Task execution (demo) |
| Metadata DB | SQLite | - | Airflow metadata (demo) |
| Webserver | Flask | - | Web UI |
| Scheduler | APScheduler | - | Cron-based scheduling |

#### Dockerfile
```dockerfile
FROM python:3.9-slim

# Install Airflow + dependencies
RUN pip install apache-airflow==2.2.5
RUN pip install kafka-python==2.0.2 psycopg2==2.9.3

# Copy DAGs and config
COPY app/ /airflow/

# Initialize Airflow DB
RUN airflow db init

# Start scheduler + webserver
CMD airflow scheduler & airflow webserver
```

#### Dependencies
```
# requirements.txt
apache-airflow==2.2.5
kafka-python==2.0.2    # Kafka producer client
psycopg2==2.9.3        # PostgreSQL driver (for production)
numpy==1.24.2          # Numerical computing
pandas==1.5.3          # Data manipulation
vnstock==0.1.4         # Vietnamese stock data (optional)
```

---

## Layer 2: Data Streaming

### Apache Kafka + Zookeeper

#### Apache Kafka 5.2.0 (Confluent Platform)

##### Vai Trò
Kafka là "event bus" của hệ thống:
- 📨 **Message Broker:** Lưu trữ và phân phối messages
- 🔁 **Decoupling:** Tách biệt producers và consumers
- 💾 **Durability:** Persist messages với configurable retention
- 🔄 **Replay:** Consumer có thể re-consume messages từ bất kỳ offset nào

##### Architecture
```
┌─────────────────────────────────────────┐
│           Kafka Broker                  │
├─────────────────────────────────────────┤
│  Topic: "demo"                          │
│  ├─ Partition 0 (Leader)                │
│  │  ├─ Segment 0: [msg0...msg999]      │
│  │  ├─ Segment 1: [msg1000...msg1999]  │
│  │  └─ ...                              │
│  └─ (No replicas - replication=1)       │
└─────────────────────────────────────────┘
```

##### Configuration (docker-compose.yaml)
```yaml
kafka:
  image: confluentinc/cp-kafka:latest
  environment:
    KAFKA_BROKER_ID: 1
    KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181

    # Listeners
    KAFKA_ADVERTISED_LISTENERS: |
      PLAINTEXT://kafka:9092,
      PLAINTEXT_HOST://localhost:29092
    KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: |
      PLAINTEXT:PLAINTEXT,
      PLAINTEXT_HOST:PLAINTEXT

    # Topic settings
    KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
    KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
    KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1

  ports:
    - "29092:29092"  # External access

  depends_on:
    - zookeeper
```

##### Topic Configuration
- **Topic Name:** `demo`
- **Partitions:** 1 (single partition - demo)
- **Replication Factor:** 1 (no replication - demo)
- **Retention:** Default 7 days
- **Compression:** None (default)
- **Message Format:** JSON UTF-8

##### Message Schema
```json
{
  "data_id": 150,        // Integer: Random price value
  "name": "BTC",         // String: Cryptocurrency symbol
  "timestamp": 1645270401 // Integer: Unix timestamp (seconds)
}
```

#### Apache Zookeeper

##### Vai Trò
Zookeeper là "configuration manager":
- 🗂️ **Metadata Storage:** Lưu cluster configuration
- 👑 **Leader Election:** Chọn partition leaders
- 🔔 **Notification:** Thông báo thay đổi cluster state
- 🔒 **Distributed Lock:** Coordination giữa brokers

##### Configuration
```yaml
zookeeper:
  image: confluentinc/cp-zookeeper:latest
  environment:
    ZOOKEEPER_CLIENT_PORT: 2181
    ZOOKEEPER_TICK_TIME: 2000  # Heartbeat interval
  ports:
    - "22181:2181"
```

##### ZooKeeper Data Structure
```
/
├── brokers/
│   ├── ids/          # Broker registrations
│   ├── topics/       # Topic metadata
│   └── seqid/        # Sequence IDs
├── consumers/        # Consumer group info
├── config/           # Configurations
└── controller        # Current controller broker
```

#### Kafka Producer (trong DAG)

```python
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['kafka:9092'],

    # Serialization
    value_serializer=lambda v: json.dumps(v).encode('utf-8'),

    # Performance tuning
    acks=1,              # Wait for leader ACK (not all replicas)
    retries=3,           # Retry failed sends
    batch_size=16384,    # Batch size in bytes
    linger_ms=10,        # Wait 10ms before sending batch

    # Compression (optional)
    compression_type='none',  # or 'gzip', 'snappy', 'lz4'
)

# Send message
future = producer.send('demo', value=message)

# Wait for confirmation (blocking)
metadata = future.get(timeout=10)
print(f"Sent to partition {metadata.partition} at offset {metadata.offset}")

# Cleanup
producer.flush()
producer.close()
```

#### Tech Stack
| Component | Technology | Version | Purpose |
|-----------|-----------|---------|---------|
| Message Broker | Apache Kafka | 5.2.0 | Event streaming |
| Coordination | Apache Zookeeper | Latest | Cluster management |
| Producer Client | kafka-python | 2.0.2 | Python Kafka client |
| Serialization | JSON | - | Message format |

---

## Layer 3: Analytics Engine

### Apache Druid 0.22.1+

#### Vai Trò
Druid là "analytical database":
- ⚡ **Fast Queries:** Sub-second aggregations on billions of rows
- 📊 **OLAP Workloads:** Multi-dimensional analytics
- ⏱️ **Time-Series Optimized:** Efficient time-based queries
- 🔥 **Real-time Ingestion:** Stream processing với low latency
- 📦 **Columnar Storage:** High compression & scan performance

#### Architecture Components

```
┌─────────────────────────────────────────────────────────┐
│                   Apache Druid Cluster                  │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────┐        ┌──────────────┐              │
│  │    Router    │───────▶│    Broker    │              │
│  │    :8888     │        │    :8082     │              │
│  └──────────────┘        └───────┬──────┘              │
│                                   │                      │
│                        ┌──────────┴──────────┐          │
│                        │                     │          │
│                        ▼                     ▼          │
│              ┌──────────────┐     ┌──────────────┐     │
│              │  Historical  │     │ MiddleManager│     │
│              │    :8083     │     │    :8091     │     │
│              └──────────────┘     └──────────────┘     │
│                        │                     │          │
│                        └──────────┬──────────┘          │
│                                   ▼                      │
│                        ┌──────────────────┐             │
│                        │   Coordinator    │             │
│                        │      :8081       │             │
│                        └──────────────────┘             │
│                                                          │
└─────────────────────────────────────────────────────────┘
            │                              │
            ▼                              ▼
    ┌──────────────┐            ┌──────────────┐
    │  PostgreSQL  │            │ Zookeeper    │
    │   Metadata   │            │   Cluster    │
    └──────────────┘            │     State    │
                                └──────────────┘
```

#### 1. Druid Router (:8888)

**Vai trò:** Unified API endpoint

**Responsibilities:**
- Route queries đến Broker nodes
- Route management requests đến Coordinator
- Load balancing across Brokers
- Authentication & authorization (nếu enable)

**Endpoints:**
- `GET /status` - Health check
- `POST /druid/v2/sql/` - SQL queries
- `POST /druid/v2/` - Native queries
- `GET /druid/coordinator/v1/` - Coordinator management API

**Configuration:**
```yaml
druid_router:
  environment:
    - druid_host=router
    - druid_service=druid/router
    - druid_plaintextPort=8888

    # Routing rules
    - druid_router_defaultBrokerServiceName=druid/broker
    - druid_router_coordinatorServiceName=druid/coordinator

    # Connection pooling
    - druid_router_http_numConnections=50
    - druid_router_http_readTimeout=PT5M
```

#### 2. Druid Broker (:8082)

**Vai trò:** Query execution engine

**Responsibilities:**
- Accept queries từ clients
- Fetch segment metadata từ PostgreSQL
- Distribute sub-queries đến Historical/MiddleManager
- Merge results từ data nodes
- Apply LIMIT, ORDER BY, final aggregations

**Query Processing:**
```
1. Parse SQL → Native Druid Query
   SELECT name, AVG(data_id) FROM demo GROUP BY name
   ↓
   {
     "queryType": "groupBy",
     "dataSource": "demo",
     "dimensions": ["name"],
     "aggregations": [{"type":"doubleSum", "name":"sum_data_id", ...}],
     ...
   }

2. Fetch Segment Metadata
   ↓ Query PostgreSQL
   Segments: [
     {id: "demo_2024-01-01T00:00:00.000Z_...", size: 1MB, location: "historical:8083"},
     {id: "demo_2024-01-01T01:00:00.000Z_...", size: 500KB, location: "middlemanager:8100"},
   ]

3. Prune Segments by Time Range
   ↓ Filter segments matching WHERE clause time range

4. Scatter Queries to Data Nodes
   ↓ Send sub-queries in parallel
   → Historical: query segments [seg1, seg2, seg3]
   → MiddleManager: query real-time segment [seg4]

5. Gather Partial Results
   ← Historical: {BTC: sum=1500, count=10}
   ← MiddleManager: {BTC: sum=300, count=2}

6. Merge & Finalize
   ↓ Combine: {BTC: avg = (1500+300)/(10+2) = 150}

7. Return to Client
   → [{name: "BTC", avg_data_id: 150}, ...]
```

**Configuration:**
```properties
# Processing
druid.processing.buffer.sizeBytes=134217728  # 128MB
druid.processing.numThreads=2
druid.processing.numMergeBuffers=2

# Caching
druid.broker.cache.useCache=true
druid.broker.cache.populateCache=true
druid.cache.type=caffeine
druid.cache.sizeInBytes=268435456  # 256MB
```

#### 3. Druid Coordinator (:8081)

**Vai trò:** Cluster management

**Responsibilities:**
- Monitor segment availability
- Assign segments đến Historical nodes
- Load balancing segments across cluster
- Drop old segments theo retention rules
- Compact small segments
- Manage replication

**Load Rules:**
```json
[
  {
    "type": "loadForever",  // Keep all data
    "tieredReplicants": {
      "_default_tier": 1    // 1 replica (no replication in demo)
    }
  }
]
```

**UI:** http://localhost:8081 - Cluster overview, segment browser

#### 4. Druid Historical (:8083)

**Vai trò:** Long-term segment storage & queries

**Responsibilities:**
- Load segments từ deep storage (/opt/shared)
- Cache segments in memory/disk
- Serve queries cho immutable segments
- Announce segments đến Zookeeper

**Segment Structure:**
```
/opt/shared/segments/demo/
└── 2024-01-01T00:00:00.000Z_2024-01-01T01:00:00.000Z/
    └── 2024-01-01T00:05:00.123Z/
        ├── 0/
        │   ├── index.zip          # Compressed columnar data
        │   │   ├── __time.column  # Timestamp column
        │   │   ├── name.column    # String column (dictionary encoded)
        │   │   ├── data_id.column # Numeric column
        │   │   └── metadata.json  # Segment metadata
        │   └── descriptor.json    # Segment descriptor
        └── version.bin
```

**Segment Format (Columnar):**
```
Row-based (traditional):
[{time:T1, name:BTC, id:150}, {time:T2, name:ETH, id:100}, ...]

Column-based (Druid):
__time:   [T1, T2, T3, T4, ...]
name:     [BTC, ETH, BTC, DOT, ...]  → Dictionary: {0:BTC, 1:ETH, 2:DOT} → [0,1,0,2,...]
data_id:  [150, 100, 155, 30, ...]
```

**Advantages:**
- ✅ High compression (dictionary encoding, run-length encoding)
- ✅ Fast scans (only read needed columns)
- ✅ Efficient aggregations (vectorized operations)

**Configuration:**
```properties
# Segment cache
druid.segmentCache.locations=[{"path":"/opt/druid/var/druid/segment-cache","maxSize":10g}]

# Processing threads
druid.processing.numThreads=2
druid.processing.buffer.sizeBytes=134217728
```

#### 5. Druid MiddleManager (:8091, :8100-8105)

**Vai trò:** Real-time ingestion & indexing

**Responsibilities:**
- Consume từ Kafka topic "demo"
- Build real-time segments in memory
- Persist segments đến deep storage
- Hand-off segments cho Historical
- Execute indexing tasks (batch ingestion)

**Ingestion Spec (Kafka Supervisor):**
```json
{
  "type": "kafka",
  "dataSchema": {
    "dataSource": "demo",
    "timestampSpec": {
      "column": "timestamp",
      "format": "posix"
    },
    "dimensionsSpec": {
      "dimensions": ["name"]
    },
    "metricsSpec": [
      {"type": "count", "name": "count"},
      {"type": "longSum", "name": "sum_data_id", "fieldName": "data_id"}
    ],
    "granularitySpec": {
      "type": "uniform",
      "segmentGranularity": "HOUR",
      "queryGranularity": "MINUTE"
    }
  },
  "tuningConfig": {
    "type": "kafka",
    "maxRowsInMemory": 100000,
    "maxBytesInMemory": 134217728,
    "intermediatePersistPeriod": "PT10M",
    "maxPendingPersists": 0
  },
  "ioConfig": {
    "topic": "demo",
    "consumerProperties": {
      "bootstrap.servers": "kafka:9092"
    },
    "taskCount": 1,
    "replicas": 1,
    "taskDuration": "PT1H"
  }
}
```

**Indexing Flow:**
```
1. Consume from Kafka
   ↓ Poll messages from topic "demo"

2. Parse & Validate
   ↓ Extract timestamp, dimensions, metrics

3. Build In-Memory Index
   ↓ Accumulate rows in heap
   ↓ When maxRowsInMemory reached → persist to disk

4. Create Segment
   ↓ After taskDuration (1 hour) or manual trigger
   ↓ Merge all persisted files
   ↓ Build final columnar segment

5. Publish to Deep Storage
   ↓ Upload segment to /opt/shared/

6. Notify Coordinator
   ↓ Register segment metadata in PostgreSQL

7. Hand-off to Historical
   ↓ Coordinator assigns segment to Historical
   ↓ Historical loads segment
   ↓ MiddleManager drops in-memory segment
```

#### Druid Extensions (app_druid/environment.env)

```properties
druid_extensions_loadList=[
  "druid-kafka-indexing-service",    # Kafka ingestion
  "druid-histogram",                  # Histogram aggregations
  "druid-datasketches",               # Approximate algorithms (HLL, Theta)
  "druid-multi-stage-query",          # Complex query engine
  "postgresql-metadata-storage"       # PostgreSQL metadata
]
```

#### Tech Stack
| Component | Technology | Version | Purpose |
|-----------|-----------|---------|---------|
| Analytics DB | Apache Druid | 0.22.1+ | OLAP engine |
| Metadata Storage | PostgreSQL | 14.1 | Segment metadata |
| Coordination | Zookeeper | Latest | Cluster state |
| Deep Storage | Local FS | - | Segment storage |
| Query Language | SQL + Native | - | Query interface |
| Ingestion | Kafka Indexing Service | - | Stream ingestion |

---

## Layer 4: Storage

### PostgreSQL 14.1 (Metadata Storage)

**Vai trò:** Metadata repository cho Druid

**Lưu trữ:**
- ✅ Segment metadata (location, size, time range)
- ✅ Supervisor & task status
- ✅ Load rules & compaction configs
- ✅ Audit logs

**Schema:**
```sql
-- Segments table
CREATE TABLE druid_segments (
    id VARCHAR(255) PRIMARY KEY,
    dataSource VARCHAR(255),
    created_date VARCHAR(255),
    start VARCHAR(255),
    "end" VARCHAR(255),
    partitioned BOOLEAN,
    version VARCHAR(255),
    used BOOLEAN,
    payload BYTEA
);

-- Supervisors table
CREATE TABLE druid_supervisors (
    id VARCHAR(255) PRIMARY KEY,
    spec_id VARCHAR(255),
    created_date VARCHAR(255),
    payload BYTEA
);
```

**Configuration:**
```yaml
postgres:
  image: postgres:14.1-alpine
  environment:
    POSTGRES_PASSWORD: FoolishPassword
    POSTGRES_USER: druid
    POSTGRES_DB: druid
  ports:
    - "5432:5432"
```

**Connection từ Druid:**
```properties
druid.metadata.storage.type=postgresql
druid.metadata.storage.connector.connectURI=jdbc:postgresql://postgres:5432/druid
druid.metadata.storage.connector.user=druid
druid.metadata.storage.connector.password=FoolishPassword
```

### Local File System (Deep Storage)

**Path:** `/opt/shared/segments/`

**Structure:**
```
/opt/shared/
├── segments/
│   └── demo/                                    # Datasource
│       ├── 2024-01-01T00:00:00.000Z_..._v1/    # Segment
│       │   └── 0/
│       │       └── index.zip
│       └── 2024-01-01T01:00:00.000Z_..._v1/
│           └── 0/
│               └── index.zip
└── task/                                        # Task working directory
    └── [temp files]
```

**Volume Mounting:**
```yaml
volumes:
  - druid_shared:/opt/shared

volumes:
  druid_shared:
```

**Production Alternatives:**
- ☁️ **AWS S3:** `druid.storage.type=s3`, `druid.storage.bucket=my-bucket`
- ☁️ **Google Cloud Storage:** `druid.storage.type=google`
- ☁️ **Azure Blob:** `druid.storage.type=azure`
- 🗄️ **HDFS:** `druid.storage.type=hdfs`

### Redis (Optional Caching)

**Vai trò:** Session storage, caching

**Use cases:**
- Airflow session management
- Superset query result caching
- Temporary data storage

**Configuration:**
```yaml
redis:
  image: redis:latest
  ports:
    - "6379:6379"
```

---

## Layer 5: Visualization

### Apache Superset 1.4.1

**Vai trò:** Business Intelligence platform

**Features:**
- 📊 **Charts:** 50+ visualization types (line, bar, pie, heatmap, etc.)
- 📈 **Dashboards:** Drag-and-drop dashboard builder
- 🔍 **SQL Lab:** Ad-hoc SQL queries với autocomplete
- 🔐 **Access Control:** Role-based permissions
- 📅 **Scheduled Reports:** Email reports theo schedule
- 🎨 **Themes:** Customizable UI themes

**Druid Connection:**
```python
# SQLAlchemy URI
SQLALCHEMY_DATABASE_URI = 'druid://broker:8082/druid/v2/sql/'

# Example query from Superset
query = """
SELECT
    name,
    TIME_FLOOR(__time, 'PT1H') as hour,
    AVG(data_id) as avg_price,
    MAX(data_id) as max_price,
    MIN(data_id) as min_price,
    COUNT(*) as count
FROM demo
WHERE __time >= CURRENT_TIMESTAMP - INTERVAL '24' HOUR
GROUP BY name, TIME_FLOOR(__time, 'PT1H')
ORDER BY hour DESC
"""
```

**Chart Types Examples:**

1. **Time Series Line Chart:**
   - X-axis: `__time` (time column)
   - Y-axis: `AVG(data_id)`
   - Group by: `name` (BTC, ETH, DOT, BTT)

2. **Bar Chart:**
   - Dimension: `name`
   - Metric: `SUM(data_id)`

3. **Pie Chart:**
   - Dimension: `name`
   - Metric: `COUNT(*)`

**Configuration:**
```yaml
superset:
  image: amancevice/superset:1.4.1
  environment:
    - SUPERSET_SECRET_KEY=your_secret_key_here
  ports:
    - "8088:8088"
  command: |
    superset db upgrade &&
    superset fab create-admin \
      --username admin \
      --password admin \
      --firstname Admin \
      --lastname User \
      --email admin@superset.com &&
    superset init &&
    superset run -h 0.0.0.0 -p 8088
```

**Access:**
- URL: http://localhost:8088
- Default credentials: admin / admin

---

## Infrastructure & DevOps

### Docker & Docker Compose

**Version:** Docker Compose 3.8

**Services:**
- ✅ 10 containers in total
- ✅ Networked via bridge network
- ✅ Persistent volumes for data
- ✅ Health checks (optional)
- ✅ Dependency management

**docker-compose.yaml Structure:**
```yaml
version: '3.8'

services:
  zookeeper: ...
  postgres: ...
  kafka: ...
  druid_coordinator: ...
  druid_broker: ...
  druid_historical: ...
  druid_middlemanager: ...
  druid_router: ...
  airflow: ...
  superset: ...

volumes:
  druid_shared:
  postgres_data:

networks:
  default:
    driver: bridge
```

**Startup Command:**
```bash
# Start all services
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f [service]

# Stop all
docker-compose down

# Remove volumes
docker-compose down -v
```

---

## Data Flow Chi Tiết

### End-to-End Flow

```
┌─────────────┐
│   Clock     │ Every minute (cron: */1 * * * *)
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────────────────────────┐
│ STEP 1: Airflow Triggers DAG                           │
├─────────────────────────────────────────────────────────┤
│ Airflow Scheduler → Create DagRun instance             │
│                   → Execute demo_func()                 │
│                   → Generate 4 messages                 │
│ Time: ~1-2 seconds                                      │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│ STEP 2: Produce to Kafka                              │
├─────────────────────────────────────────────────────────┤
│ KafkaProducer → Serialize JSON to bytes                │
│               → Send to topic "demo"                    │
│               → Kafka appends to partition log          │
│ Latency: <50ms                                          │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│ STEP 3: Druid Ingestion (Real-time)                   │
├─────────────────────────────────────────────────────────┤
│ MiddleManager → Poll Kafka (fetch.min.bytes=1)         │
│               → Parse JSON & extract fields             │
│               → Add to in-memory segment                │
│               → Build columnar index                    │
│ Latency: <500ms (queryable immediately)                 │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│ STEP 4: Segment Handoff (Every hour)                  │
├─────────────────────────────────────────────────────────┤
│ MiddleManager → Finalize segment                       │
│               → Persist to /opt/shared/                 │
│               → Update PostgreSQL metadata              │
│ Coordinator   → Assign segment to Historical           │
│ Historical    → Load segment from deep storage         │
│ MiddleManager → Drop in-memory segment                 │
│ Time: ~10-30 seconds                                    │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│ STEP 5: Query Execution                                │
├─────────────────────────────────────────────────────────┤
│ User → Superset → Router → Broker                      │
│ Broker → Fetch segment metadata from PostgreSQL        │
│        → Query Historical (old segments)                │
│        → Query MiddleManager (recent data)              │
│        → Merge results                                  │
│ Router → Return to Superset                             │
│ Superset → Render chart                                 │
│ Latency: 50ms-2s depending on query complexity          │
└─────────────────────────────────────────────────────────┘
```

### Data Latency Breakdown

| Stage | Latency | Description |
|-------|---------|-------------|
| Airflow schedule trigger | 0-60s | Cron precision (1 minute granularity) |
| Message generation | 1-2s | Python execution time |
| Kafka produce | <50ms | Network + broker write |
| Druid consume | <500ms | Poll interval + parsing |
| **Query latency (real-time)** | **<1s** | **End-to-end from produce to queryable** |
| Segment handoff | 1-60min | Based on taskDuration config |
| Query execution (simple) | <100ms | Aggregation on indexed data |
| Query execution (complex) | 100ms-2s | Multi-table joins, large scans |

---

## Production Best Practices

### 1. Scalability

**Airflow:**
- ❌ SequentialExecutor (demo) → ✅ CeleryExecutor / KubernetesExecutor
- ❌ SQLite metadata → ✅ PostgreSQL / MySQL
- ✅ Add Celery workers để parallel task execution
- ✅ Redis/RabbitMQ làm message broker cho Celery

**Kafka:**
- ❌ Single broker → ✅ Cluster with 3+ brokers
- ❌ Replication factor = 1 → ✅ Replication = 3
- ✅ Multiple partitions cho high throughput topics
- ✅ Enable compression (lz4, snappy)

**Druid:**
- ✅ Multiple Historical nodes (scale horizontally)
- ✅ Multiple MiddleManager nodes
- ✅ Multiple Broker nodes behind load balancer
- ❌ Local storage → ✅ S3/GCS/HDFS
- ✅ Tiered storage (hot/cold data)

### 2. High Availability

**Infrastructure:**
- ✅ Zookeeper ensemble (3 hoặc 5 nodes)
- ✅ PostgreSQL replication (master-slave)
- ✅ Kafka replication across availability zones
- ✅ Load balancers cho Druid Router & Broker

**Monitoring:**
- ✅ Prometheus + Grafana
- ✅ ELK stack (Elasticsearch, Logstash, Kibana)
- ✅ Alerting (PagerDuty, Slack)

### 3. Security

**Authentication:**
- ✅ Airflow: LDAP / OAuth2
- ✅ Superset: LDAP / SAML
- ✅ Kafka: SASL/SCRAM or SASL/PLAIN
- ✅ Druid: Basic Auth / Kerberos

**Encryption:**
- ✅ SSL/TLS cho tất cả connections
- ✅ Encrypt data at rest (disk encryption)
- ✅ Kafka SSL encryption

**Network:**
- ✅ VPC / Private subnets
- ✅ Security groups / Firewall rules
- ✅ Bastion hosts cho admin access

### 4. Performance Tuning

**Druid:**
```properties
# Increase processing threads
druid.processing.numThreads=8  # = CPU cores - 1

# Increase buffer size
druid.processing.buffer.sizeBytes=536870912  # 512MB

# Enable caching
druid.broker.cache.sizeInBytes=2147483648  # 2GB
druid.historical.cache.sizeInBytes=2147483648

# Query optimization
druid.query.groupBy.maxMergingDictionarySize=100000000
druid.query.groupBy.maxOnDiskStorage=10737418240  # 10GB
```

**Kafka:**
```properties
# Producer
batch.size=32768
linger.ms=10
compression.type=lz4
acks=all

# Consumer
fetch.min.bytes=1048576  # 1MB
max.poll.records=500
```

### 5. Monitoring Metrics

**Airflow:**
- ✅ DAG run duration
- ✅ Task success/failure rate
- ✅ Scheduler lag
- ✅ Executor queue size

**Kafka:**
- ✅ Broker CPU/Memory/Disk usage
- ✅ Partition lag
- ✅ Producer/Consumer throughput
- ✅ Under-replicated partitions

**Druid:**
- ✅ Query latency (p50, p95, p99)
- ✅ Segment count & size
- ✅ Ingestion rate
- ✅ Query cache hit rate
- ✅ JVM heap usage

---

## Kết Luận

Hệ thống Real-Time Analytics này là một ví dụ hoàn chỉnh về:

✅ **Lambda Architecture** - Kết hợp speed layer (real-time) và batch layer (historical)
✅ **Event-Driven Architecture** - Kafka làm event bus
✅ **Microservices** - Mỗi component độc lập, scale riêng
✅ **OLAP Analytics** - Druid optimized cho analytical queries
✅ **Workflow Orchestration** - Airflow quản lý dependencies
✅ **Data Visualization** - Superset cho dashboards

**Điểm mạnh:**
- ⚡ Low latency (<1s từ ingestion đến queryable)
- 📈 Horizontal scalability
- 🔄 Fault-tolerant với Kafka replay
- 💾 Efficient columnar storage
- 🎯 Production-ready architecture (với modifications)

**Production Checklist:**
- [ ] Migrate to CeleryExecutor (Airflow)
- [ ] Kafka cluster với replication ≥ 3
- [ ] PostgreSQL replication
- [ ] S3/GCS deep storage (Druid)
- [ ] Multiple Druid nodes
- [ ] SSL/TLS encryption
- [ ] Authentication & authorization
- [ ] Monitoring stack (Prometheus/Grafana)
- [ ] Backup & disaster recovery plan
- [ ] Auto-scaling (Kubernetes/ECS)
