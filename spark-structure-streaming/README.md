# Spark Structured Streaming

PySpark + Jupyter + Kafka — local setup and run for learning **Spark Structured Streaming**.

---

## What This Project Covers

| Component | Purpose |
|-----------|---------|
| **Docker Compose** | Runs Zookeeper, Kafka, PySpark Jupyter Lab, PostgreSQL, and SQLPad with one command |
| **Jupyter Lab** | Run PySpark batch and streaming code (socket and Kafka sources) |
| **Kafka** | Create topics and produce/consume messages for streaming examples |
| **TCP socket (ncat)** | Simple streaming source for your first streaming job |
| **PostgreSQL + SQLPad** | Optional: query and visualize data (SQLPad at `http://localhost:3000`) |

**Flow:** Start the stack → Get Jupyter URL/token → Open Jupyter in the browser → Create a Kafka topic → Start a data source (ncat or Kafka producer) → Run the streaming notebooks end to end.

---

## Prerequisites

- **Docker** and **Docker Compose** installed
- Ports **8888**, **9092**, **29092**, **3000** free (or adjust mappings in `docker-compose.yml`).  
  If port 3000 is in use, stop the other process or map SQLPad to another port.

---

## Step 1 — Start the Stack

From the project root (or wherever your `docker-compose.yml` lives, e.g. `docker-images-master/pyspark-jupyter-kafka/`):

```bash
docker compose up
```

Wait until all containers are up. If you see "address already in use" for port 3000, free that port or change the SQLPad port in `docker-compose.yml`, then run `docker compose up` again.

---

## Step 2 — Get Jupyter URL and Token

In a new terminal:

```bash
docker exec -it ed-pyspark-jupyter-lab /bin/bash
```

Then inside the container:

```bash
jupyter server list
```

Copy the URL that looks like: `http://....:8888/?token=...` (the long string after `token=`). You will use it in the next step.

---

## Step 3 — Open Jupyter in Your Browser

1. Go to **http://localhost:8888/lab**
2. When prompted for a password or token, paste the token from the `jupyter server list` output.
3. You can now create notebooks and run PySpark code.

---

## Step 4 — Run Your First Streaming Job (TCP Socket)

### 4.1 Install ncat (if needed)

Inside the Jupyter container:

```bash
docker exec -it ed-pyspark-jupyter-lab /bin/bash
apt-get update && apt-get install -y nmap   # ncat is provided by nmap
ncat -v   # optional: verify ncat is available, then Ctrl+C to exit
```

### 4.2 Start a socket data source (separate terminal)

```bash
docker exec -it ed-pyspark-jupyter-lab /bin/bash -c "ncat -l 9999"
```

- **`docker exec -it`** — run a command inside the running container  
- **`ed-pyspark-jupyter-lab`** — Jupyter container name  
- **`ncat -l 9999`** — TCP listener on port 9999 inside the container  

Type lines of text and press Enter. Each line is one micro-batch for the streaming job. Leave this terminal running.

### 4.3 In Jupyter — create a new notebook and run:

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("SocketStream").getOrCreate()

lines = (
    spark.readStream
    .format("socket")
    .option("host", "localhost")
    .option("port", 9999)
    .load()
)

query = lines.writeStream.outputMode("append").format("console").start()
query.awaitTermination()
```

### 4.4 End-to-end

With `ncat -l 9999` running and the notebook cell running, type in the ncat terminal and watch the same lines appear in the Jupyter console output.

**To stop:** Interrupt the notebook (e.g. **Kernel → Interrupt**), then stop ncat with `Ctrl+C`.

---

## Step 5 — Run a Kafka Streaming Job

### 5.1 Create a Kafka topic

From your host (use the Kafka container name from your compose setup, e.g. `ed-kafka`):

```bash
docker exec -it ed-kafka /bin/bash
```

Inside the Kafka container, create and list topics. Use **`localhost:29092`** when running commands inside the Kafka container:

```bash
kafka-topics --create --topic device-data --bootstrap-server localhost:29092
kafka-topics --list --bootstrap-server localhost:29092
```

Exit the Kafka container when done (`exit`).

### 5.2 Produce messages to the topic

In a terminal, start the console producer (again from inside the Kafka container, use `localhost:29092`):

```bash
docker exec -it ed-kafka /bin/bash -c "kafka-console-producer --topic device-data --bootstrap-server localhost:29092"
```

Then type JSON messages and press Enter after each. Example:

```json
{"eventId": "ba2ea9f4-a5d9-434e-8e4d-1c80c2d4b456", "eventOffset": 10000, "eventPublisher": "device", "customerId": "CI00119", "data": {"devices": []}, "eventTime": "2023-01-05 11:13:53.643364"}
```

```json
{"eventId": "e3cb26d3-41b2-49a2-84f3-0156ed8d7502", "eventOffset": 10001, "eventPublisher": "device", "customerId": "CI00103", "data": {"devices": [{"deviceId": "D001", "temperature": 15, "measure": "C", "status": "ERROR"}, {"deviceId": "D002", "temperature": 16, "measure": "C", "status": "SUCCESS"}]}, "eventTime": "2023-01-05 11:13:53.643364"}
```

### 5.3 In Jupyter — read from Kafka

From **inside the Jupyter container**, use the Kafka service name and port (e.g. **`ed-kafka:29092`** as in `read_kafka.ipynb`):

```python
from pyspark.sql import SparkSession

spark = (
    SparkSession.builder
    .appName("Streaming from Kafka")
    .config("spark.jars.packages", "org.apache.spark:spark-sql-kafka-0-10_2.12:3.3.0")
    .getOrCreate()
)

kafka_df = (
    spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "ed-kafka:29092")
    .option("subscribe", "device-data")
    .option("startingOffsets", "earliest")
    .load()
)

# Then add your transformations and writeStream (e.g. .writeStream.format("console").start())
```

**Note:** Use **`ed-kafka:29092`** in Jupyter (Docker network hostname). Use **`localhost:29092`** when running `kafka-topics` or `kafka-console-producer` inside the Kafka container. If your compose uses port **9092** for the broker, use that port consistently in both places.

---

## Project Notebooks

| Notebook | Description |
|----------|-------------|
| `reading_from_socket.ipynb` | Socket streaming example |
| `read_kafka.ipynb` | Kafka streaming with device-data topic and JSON parsing |
| `flatten_json_file.ipynb` | Batch JSON flattening (e.g. for device payloads) |

---

## Troubleshooting

- **Port already in use:** Change the conflicting port in `docker-compose.yml` or stop the process using it.
- **Cannot connect to Kafka from Jupyter:** Ensure bootstrap servers use the Kafka service name (e.g. `ed-kafka:29092`) and that the port matches your compose Kafka configuration.
- **Container name differs:** If your Kafka container is not `ed-kafka`, replace it in all `docker exec` commands and in the Jupyter `kafka.bootstrap.servers` option. List containers with `docker ps`.
