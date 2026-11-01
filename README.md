# HamAmooz — Kubernetes Management Platform

A small web application for managing Kubernetes resources. I built the backend with Django REST Framework and the frontend with React, then deployed the application on a two-node K3s cluster.

## What it does

- Register clusters and verify Kubernetes API connectivity
- View and manage namespaces and application Deployments
- Keep inactive applications at zero replicas
- Run and schedule backups through Celery
- Use viewer, staff, and administrator permissions for management actions
- Show workload status and application metrics in the console and Grafana

## Architecture

```text
Browser -> React / Nginx -> Django API -> Kubernetes API
                                      |-> Redis
                                      |-> Celery worker / beat

Django + Celery metrics -> VMAgent -> VMSingle -> VMAuth -> Grafana
```

Redis caches workload status; Celery handles scheduled and background work. VictoriaMetrics collects internal application metrics, including Kubernetes operations, latency, and backup outcomes. The public web route does not expose `/metrics`.

## Run locally

The Docker Compose setup runs the application and supporting development services. It expects an external Docker network named `minikube`; create it locally if it does not exist:

```bash
docker network inspect minikube >/dev/null 2>&1 || docker network create minikube
docker compose up --build -d
docker compose exec backend python manage.py createsuperuser
```

The web interface and API are available at `http://localhost:8000`; Grafana is at `http://localhost:3000`. To stop the stack, use `docker compose down`.

## Development and tests

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python manage.py migrate
python manage.py test
```

For the frontend:

```bash
cd frontend
npm ci
npm run dev
npm run build
```

## Kubernetes deployment

The deployment manifests target release `hamamooz-v1.4.1` on K3s. The application uses Traefik, Redis, Celery, VictoriaMetrics, and Grafana. See [the deployment guide](docs/REMOTE_DEPLOYMENT_GUIDE.md) for credentials, image loading, resource manifests, and verification commands.

Useful post-deployment checks:

```bash
kubectl get pods,svc,ingress -n hemmasian
kubectl get vmsingle,vmagent,vmauth,vmservicescrape -n monitoring-hamamooz-task
```

## Security and limits

Cluster tokens are encrypted at rest and are not returned by the API. Management endpoints require authentication, and cluster-changing operations are restricted to staff or administrator accounts. The reference remote deployment uses HTTP; add TLS before exposing it outside a trusted network. Development credentials from Docker Compose must not be reused in production.

The deployment and architecture are intentionally small rather than a highly available multi-cluster platform.
