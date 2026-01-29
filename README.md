# Spark-Structured-Streaming

# PySpark Jupyter + Kafka — Local Setup & Run

## What This Project Covers

| Topic / Component | Purpose |
|-------------------|--------|
| **Docker Compose** | Runs Zookeeper, Kafka, PySpark Jupyter Lab, PostgreSQL, SQLPad in one command |
| **Jupyter Lab** | Run PySpark batch and streaming code (sockets, Kafka) |
| **Kafka** | Create topics and produce/consume messages for streaming examples |
| **TCP socket (ncat)** | Simple streaming source for your first streaming job |
| **PostgreSQL + SQLPad** | Optional: query and visualize data (SQLPad at `http://localhost:3000`) |

**Flow:** Start stack → Get Jupyter URL/token → Open Jupyter in browser → Create Kafka topic → Start data source (ncat or Kafka producer) → Run streaming notebook end to end.

---

## Prerequisites

- **Docker** and **Docker Compose** installed
- Ports **8888**, **9092**, **3000** free (or change mappings in `docker-compose.yml`). 
  If port 3000 is in use, stop the other process or map SQLPad to another port.

---

## Step 1 — Start the Stack

From the project root (e.g. `docker-images-master/pyspark-jupyter-kafka/`):
```
docker compose up
```
Wait until all containers are up. If you see "address already in use" for port 3000, 
free that port or change the SQLPad port in docker-compose.yml, 
then run docker compose up again.

## Step 2 — Get Jupyter URL and Token
In a new terminal:
```
docker exec -it ed-pyspark-jupyter-lab /bin/bash
```
Opens an interactive bash shell inside the running Docker container named ed-pyspark-jupyter-lab.
```
jupyter server list
```
Lists all running Jupyter servers inside the container.
Shows the URL + token you need to open Jupyter in your browse

Copy the URL that looks like: http://406f977e69e6:8888/?token=.... You will use it in the next step.

## Step 3 — Open Jupyter in Your Browser
In the browser, go to: http://localhost:8888/lab
When prompted for a password/token, paste the token from the jupyter server list output (the long string after token=).
You can now create notebooks and run PySpark code.

## Step 4 — Run Your First Streaming Job (TCP Socket)
4.1 Install ncat (if not already available)
Inside the Jupyter container:

```
docker exec -it ed-pyspark-jupyter-lab /bin/bash
apt-get update
apt-get install -y ncat
ncat -v
```
nmap provides ncat. ncat -v checks that ncat is available (optional). Exit with Ctrl+C if you only ran it to verify.

4.2 Start a socket data source (separate terminal)
```
docker exec -it ed-pyspark-jupyter-lab /bin/bash -c "ncat -l 9999"
```
What it’s doing 

docker exec -it → run a command inside a running container

ed-pyspark-jupyter-lab → container name

/bin/bash -c → execute a shell command

ncat -l 9999 → start Netcat in listen mode on port 9999

In short
 It opens a TCP listener on port 9999 inside the container and waits for incoming connections.


Type lines of text and press Enter. Each line is one micro-batch of input for the streaming job. Leave this running.

## 4.2 In Jupyter, create a new notebook and run:
```
   from pyspark.sql import SparkSession
   spark = SparkSession.builder.appName("SocketStream").getOrCreate()
   lines = spark.readStream.format("socket").option("host", "localhost").option("port", 9999).load()
   query = lines.writeStream.outputMode("append").format("console").start()
   query.awaitTermination()
```

## 4.3 End-to-end:
With ncat -l 9999 running and the notebook cell running, type in the ncat terminal and watch the same lines appear in the Jupyter console output.
To stop: interrupt the notebook cell (e.g. Kernel → Interrupt), then stop ncat (Ctrl+C).

## Step 5 — Run a Kafka Streaming Job
5.1 Create a topic (from host or another terminal):
```
docker exec -it ed-kafka /bin/bash
kafka-topics --create --topic test-topic --bootstrap-server ed-kafka:9092
kafka-topics --list --bootstrap-server ed-kafka:9092
```
## 5.2 In Jupyter
Use Kafka as source (read from test-topic, bootstrap ed-kafka:9092) and write to console or another sink. Same pattern: readStream → transformations → writeStream (e.g. format("console")).

## 5.3 Produce messages
Produce messages to test-topic with kafka-console-producer inside the Kafka container so your streaming job can consume them.
