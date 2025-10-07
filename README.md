# 🐳 Docker Swarm Monitoring Lab

This lab provides a hands-on environment to deploy and monitor microservices using **Docker Swarm**, **Prometheus**, **Grafana**, and **VictoriaMetrics**. It simulates a real-world deployment with separated service and monitoring stacks.

---

## 📁 Project Structure

```
swarmproject/
├── backapp/                     # Service stack
│   ├── app/                    # Flask app with Redis + MongoDB
│   ├── docker-compose.yml
│   └── nginx/
├── monitoring/                  # Monitoring stack
│   ├── prometheus/
│   ├── grafana/
│   └── docker-compose.yml
└── README.md
```

---

## 🚀 Features

- **Flask + Redis + MongoDB + Nginx** service stack
- **Prometheus + Grafana + VictoriaMetrics + Node Exporter** monitoring stack
- `/metrics` endpoint exposed using `prometheus_client`
- Grafana dashboards (manual or provisioned)
- Optional: Deploy Portainer for a Swarm UI

---

## ✅ Prerequisites

- Docker installed with Swarm initialized:

  ```bash
  docker --version
  docker swarm init
  ```

- External overlay networks must be created once:

  ```bash
  docker network create -d overlay backend
  docker network create -d overlay monitoring
  ```

---

## 📦 1. Clone the Repository

```bash
git clone https://github.com/MaxMukhtarov/swarmproject.git
cd swarmproject
```

---

## 🔧 2. Deploy the Stacks

### A. **Service Stack** (Flask + Redis + Mongo + Nginx)

Ensure the `backapp/docker-compose.yml` is using your image:

```yaml
services:
  web:
    image: deoauth2/backapp-web:latest
```

Then deploy:

```bash
docker stack deploy -c backapp/docker-compose.yml backapp
```

### B. **Monitoring Stack** (Prometheus + Grafana + VictoriaMetrics + Node Exporter)

```bash
docker stack deploy -c monitoring/docker-compose.yml monitoring
```

---

## 🌐 3. Access the Services

| Component         | URL                        | Notes                                |
|------------------|----------------------------|--------------------------------------|
| Nginx (Web app)   | http://localhost           | Proxies to Flask app                 |
| Flask Metrics     | http://localhost:5050/metrics | Prometheus scrapes this           |
| Prometheus        | http://localhost:9090      | Query metrics, check targets         |
| Grafana           | http://localhost:3000      | `admin / admin` login                |
| VictoriaMetrics   | http://localhost:8428      | Long-term metrics storage (optional) |

---

## 📊 4. Set Up Grafana

1. Go to [http://localhost:3000](http://localhost:3000)
2. Login: `admin` / `admin`
3. Add Prometheus data source:
   - **Name**: Prometheus
   - **URL**: `http://prometheus:9090`
4. Save and test the connection
5. Create dashboards using:
   - `http_requests_total`
   - Node Exporter metrics (CPU, Memory, Disk, etc.)

---

## 📈 5. Scale & Observe

Try scaling your web service:

```bash
docker service scale backapp_web=3
```

Simulate a failure:

```bash
docker service scale backapp_web=0
```

Watch Grafana dashboards and Prometheus behavior.

---

## 🧭 (Optional) Launch Portainer UI

Portainer provides a browser-based UI to manage Docker Swarm.

To deploy it:

```bash
docker service create --name portainer --publish 1000:9000 --constraint "node.role == manager" --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock --mount type=volume,src=portainer_data,dst=/data portainer/portainer-ce:latest
```

- Open in browser: [http://localhost:1000](http://localhost:1000)
- Create admin user
- Select **"Docker"** as the environment
- Browse your Swarm cluster visually

---

## 🧹 Teardown

To stop everything:

```bash
docker stack rm backapp
docker stack rm monitoring
docker service rm portainer
```

To clean up volumes and networks (optional):

```bash
docker network rm backend monitoring
docker volume rm portainer_data
```

---

## 📌 Notes

- This setup is intended for **single-node Docker Swarm**.
- All services are connected via the `backend` and `monitoring` overlay networks.
- You can extend this with:
  - Prometheus alerts
  - Pre-provisioned Grafana dashboards
  - GitHub Actions for CI/CD
  - TLS using Traefik or Caddy

---

## 📬 Feedback

Pull requests and issues welcome!
