## Realtime Data Streaming (Lightweight Test Project)

This is a simple example project that shows how to build a realtime data pipeline using Kafka, Apache Spark, Cassandra, and Apache Airflow. Everything runs in Docker using docker-compose for easy local testing.

---

## What is Included

- **Kafka + Zookeeper** — message queue for streaming data
- **Spark Structured Streaming** — reads from Kafka and writes to Cassandra
- **Cassandra** — stores the processed user data
- **Airflow** — runs a scheduled job that produces fake user data to Kafka
- **Control Center / Schema Registry** — optional tools for managing Kafka

## How It Works

1. Airflow DAG fetches fake user data from an API and pushes it to Kafka (`users_created` topic).
2. Spark Streaming job reads from Kafka, processes the JSON messages, and writes them into a Cassandra table.
3. Cassandra stores the final records for further querying or analysis.

## Architecture (quick)

1. Airflow DAG periodically fetches data from an external API and produces JSON messages to Kafka topic `users_created`.
2. Spark Structured Streaming job reads the Kafka topic, parses JSON and writes rows into Cassandra.
3. Cassandra stores the final records.

## Folder Structure

```
├── dags/                # Airflow DAGs
│   └── kafka_stream.py
├── spark_stream.py      # Spark streaming script
├── docker-compose.yml   # Main stack setup
├── requirements.txt     # Python dependencies
└── script/
    └── entrypoint.sh    # Airflow startup script
```

## Quick start (run everything with Docker Compose)

1. From repository root bring up the stack:

```bash
docker compose up -d
```

2. Watch the logs if you need to inspect startup:

```bash
docker compose logs -f webserver   # Airflow web UI logs
docker compose logs -f broker      # Kafka broker logs
```

3. Airflow web UI is exposed on http://localhost:8080

- Default admin user (created by the entrypoint):
  - username: `admin`
  - password: `admin`

4. Control Center (Confluent) is available on http://localhost:9021 (if included in your compose). Schema Registry on port 8081.

Notes:

- The Airflow `webserver` container mounts `./dags` so your DAG (`dags/kafka_stream.py`) will be loaded automatically.
- The webserver entrypoint uses `script/entrypoint.sh` to install `requirements.txt` inside the container and create the Airflow DB and an admin user.

## Run Spark streaming locally (optional)

If you want to run the Spark streaming script outside Docker (for development), install Python deps and run `spark_stream.py`:

1. Create virtualenv and install requirements (may be large):

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
# You may need to install pyspark separately that matches the cluster version (e.g. pyspark==3.5.0)
```

2. Ensure Kafka and Cassandra are reachable. If using the docker-compose stack from above, the broker is reachable at `localhost:9092` and Cassandra at `localhost:9042`.

3. Run the Spark script:

```bash
python spark_stream.py
```

Notes: the script configures Spark with additional packages (Cassandra connector and Spark-Kafka integration). When running outside Docker you may need to set `spark.jars.packages` or use a matching `pyspark` distribution.

## Key configuration details

- Kafka topic used: `users_created` (producer in `dags/kafka_stream.py`)
- Cassandra keyspace: `spark_streams`; table: `created_users` (created by `spark_stream.py`).
- Airflow scheduling: `dags/kafka_stream.py` uses schedule `@daily` by default.

## Troubleshooting

- If Airflow web UI doesn't come up: check `docker compose logs webserver` and confirm the DB is initialized. The entrypoint script installs requirements and runs `airflow db init` / creates admin user.
- If Kafka producer can't connect: confirm the broker is healthy (check `docker compose ps` and logs) and try connecting to `localhost:9092`.
- If Spark can't write to Cassandra: verify `cassandra_db` is healthy and reachable on `9042`. Check connector package versions for compatibility.

## Development notes & next steps

- You can add more DAGs into `dags/` — they are mounted into the Airflow container.
- Consider switching Airflow to a non-Sequential executor for production/testing with multiple workers (this compose uses Sequential executor in environment variables).
- Add CI checks for linting and tests if you plan to extend this project.
