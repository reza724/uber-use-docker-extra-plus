# GeoRide — a small Uber on Docker Compose

GeoRide is a tiny ride-hailing app. A rider asks for a ride, and the system finds the nearest driver and assigns them to the trip. Next to the app there is a monitoring stack that collects metrics and logs.

Everything runs with one `docker-compose.yml`: 14 containers on one network called `georide-net`.

## How a ride request works

1. You send `POST /request_ride` to **nginx**, which passes it to **api**.
2. **api** saves the trip in **postgres** with status `requested`, puts a message in the **rabbitmq** queue `ride_requests`, and answers at once with `202` and a `trip_id`.
3. **matcher** takes the message from the queue and asks **location** for the nearest driver.
4. **location** searches **redis** (GEO commands) and returns the driver's name.
5. **matcher** sets the trip to `assigned` in postgres (or `cancelled` if no driver is found).
6. In the background, **simulator-alpha** sends the driver's position to location every 5 seconds.

The queue lets the api answer fast. If matcher is slow or down, the message waits in the queue and is not lost.

## Services

**App services (built from this repo)**

| Service | What it does |
|---|---|
| api | Flask web API: `POST /register`, `POST /request_ride`, `GET /trip/<id>`. |
| matcher | Background worker. Reads ride requests from RabbitMQ and assigns a driver. |
| location | Go service. Stores driver positions in Redis and finds the nearest one. Exposes metrics on port 9091. |
| simulator-alpha | A fake driver (`Driver-Alpha`) that sends a random position every 5 seconds. |

**Core services**

| Service | What it does |
|---|---|
| postgres | Main database. Tables `users` and `trips`. `init.sql` creates them and adds 3 drivers. |
| rabbitmq | Message queue between api and matcher. Web UI on port 15672. |
| redis | In-memory store for live driver positions. |
| nginx | Reverse proxy. The only entry point to the app; forwards requests to api. |
| prometheus | Collects metrics every 15 seconds from node-exporter, cadvisor and location. |
| loki | Stores logs, labeled by service and container. |
| promtail | Reads the logs of all project containers from Docker and sends them to loki. |

**Monitoring helpers**

| Service | What it does |
|---|---|
| node-exporter | Metrics for the host machine: CPU, memory, disk, network. |
| cadvisor | Metrics for each container. |
| grafana | Dashboards. Reads metrics from prometheus and logs from loki. |

## Run it

```bash
docker compose up -d
docker compose ps
```

| What | Address | Login |
|---|---|---|
| App (through nginx) | http://localhost:8000 | — |
| Grafana | http://localhost:3000 | admin / admin |
| Prometheus | http://localhost:9090 | — |
| RabbitMQ | http://localhost:15672 | guest / guest |

In Grafana, add two data sources: Prometheus at `http://prometheus:9090` and Loki at `http://loki:3100`.

### Try it

```bash
curl -X POST localhost:8000/register -H "Content-Type: application/json" -d '{"name":"Carlos","type":"rider"}'
curl -X POST localhost:8000/request_ride -H "Content-Type: application/json" -d '{"rider_id":4,"latitude":34.05,"longitude":-118.24}'
curl localhost:8000/trip/1
```

On a fresh database the first rider gets `user_id` 4, because three drivers already exist. After a few seconds the trip shows `{"driver_id":1,"status":"assigned"}`.

## What was broken and how it was fixed

| # | Problem | What happened | Fix |
|---|---|---|---|
| 1 | No `.env` file | postgres exited: superuser password not set. | Added `.env` with `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`. |
| 2 | Project name came from the folder name | promtail only reads project `georide`, so no logs reached loki. | Replaced the old `version:` line with `name: georide`. |
| 3 | `./loki/data` was mounted from the host | loki runs as user 10001 and got `permission denied`. | Used a Docker volume `loki_data` and removed the old data folder. |
| 4 | Host port 8080 was already in use | cadvisor could not start. | Removed the host port. Prometheus reaches cadvisor inside the network. |
| 5 | Host port 80 was used by apache2 | nginx could not start. | Mapped nginx to `8000:80`. |
| 6 | matcher waits only 15 s for RabbitMQ | If RabbitMQ is slower, matcher crashes and stays down. | Added `restart: on-failure`. |
| 7 | `import psycopg2.extras` was missing in `app.py` | `GET /trip/<id>` returned `500`. | Added the import. |
| 8 | Driver search radius was 5 km, but the simulator moves the driver up to ~15 km away | 3 of 5 test trips were `cancelled`. | Radius set to 20 km. After the fix, 5 of 5 trips were `assigned`. |

Problems 4 and 5 depend on the machine. If other ports are busy on your computer, change only the left number in `ports`.

## CI/CD

`.github/workflows/docker.yml` runs on every push:

1. Check out the code.
2. Log in to Docker Hub with the repository secrets `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN`.
3. Build the four app images with `docker compose build`.
4. Push them to Docker Hub as `rezaslt20/georide-api`, `georide-location`, `georide-matcher` and `georide-simulator-alpha`.

Images: https://hub.docker.com/u/rezaslt20
