# Monitoring Setup —  Prometheus + Loki + Grafana

## Architecture

```
# metrics
Fastapi/Django app  →  /metrics endpoint  →  Prometheus (scrapes every 5s)  →  Grafana

#logs
FastAPI/Django app  →  stdout JSON logs   →  Promtail (tails Docker logs)   →  Loki  →  Grafana
```

## 📁 File Structure

```
├── .env.docker                        # docker env file
├── docker-compose.yml                 # Main docker-compose file
├── docker-compose.monitoring.yml         # docker-compose file for monitoring
├── monitoring/                         # Monitoring stack
│   ├── datasources.yaml                # Grafana datasources
│   ├── loki-config.yaml                 # Loki logging config
│   ├── prometheus.yml                   # Prometheus config
│   └── promtail-config.yaml             # Promtail config
├── post/                                # Post service
│   ├── backend/                          # Post backend (FastAPI)
│   │   ├── .env                          # Environment variables
│   │   ├── Dockerfile                     # Backend Dockerfile
│   │   ├── requirements.txt                # Python dependencies
│   │   └── app/                            # Application code
│   │       ├── main.py                      # Entry point
│   │       ├── core/                         # Core functionality
│   │       ├── db/                            # Database models
│   │       ├── internal/                      # Internal logic
│   │       ├── models/                        # Pydantic models
│   │       ├── routers/                       # API routes
│   │       ├── schemas/                       # Data schemas
│   │       └── tests/                          # Tests
│   └── frontend/                           # Post frontend (empty)
└── user/                                 # User service
    ├── backend/                           # User backend (Django)
    │   ├── .env                             # Environment variables
    │   ├── Dockerfile                        # Backend Dockerfile
    │   ├── manage.py                          # Django manage.py
    │   ├── requirements.txt                   # Python dependencies
    │   ├── apps/                              # Django apps
    │   │   ├── base/                            # Base app
    │   │   ├── communications/                   # Communications app
    │   │   ├── profiles/                         # Profiles app
    │   │   ├── relationships/                    # Relationships app
    │   │   └── users/                            # Users app
    │   ├── logs/                               # Log files
    │   └── settings/                           # Django settings
    │       ├── asgi.py                           # ASGI config
    │       └── settings/                         # Settings modules
    │           ├── base.py                        # Base settings
    │           ├── dev.py                          # Development settings
    │           └── prod.py                         # Production settings
    └── frontend/                            # User frontend (empty)
```

> [!NOTE]
> ###### please note that most files are not shown in this structure and only main files required for docker and monitoring are displayed. 

> [!NOTE]
> ###### if you see the file naming is inconvenient feel free to rename them but be careful to also change their names in docker and config files


## Setup Steps

### 1. Install docker
install docker from [docker](https://www.docker.com/get-started)


### 2. make sure Fastapi and Django are running
start docker compose file for post and user services with
```
docker compose -f ./docker-compose.yml up
```


### 3. Start monitoring services
```bash
docker compose -f docker-compose.monitoring.yml up
```

### 5. Open Grafana
- URL: http://localhost:3000
- Login: admin / admin
- Prometheus and Loki are auto-connected

>[!TIP]
>grafana asks for changing the password after first login, but if you want to change the credentials before starting the container, edit these two lines in docker-compose.monitoring.yml file
```
- GF_SECURITY_ADMIN_USER=admin
- GF_SECURITY_ADMIN_PASSWORD=admin
```

## What You Get

### Logs (Loki → Grafana)
Query examples in Grafana Explore tab:
```logql
# All logs from fastapi
{service="post"}

# Only errors
{service="post"} | json | level="error"

# Specific user's activity  
{service="post"} | json | user_id="abc-123"

# Specific IP address activity
{service="post"} | json | ip="192.168.1.1"

# All failed logins
{service="post"} | json | event="login_failed"

# Slow requests (>500ms)
{service="post"} | json | duration_ms > 500
```

### Metrics (Prometheus → Grafana)
Useful PromQL queries for dashboards:
```promql
# Request rate per endpoint
rate(http_requests_total[5m])

# 95th percentile latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Error rate
rate(http_requests_total{status=~"5.."}[5m])

# Total posts created
posts_created_total

# bookmark rates
rate(bookmarks_total[5m])

```

## Sample Log Output
Every request automatically produces structured logs like:
```json
{"event": "request_started",  "ip": "192.168.1.50", "user_id": "abc-123", "method": "POST", "path": "/posts/", "timestamp": "2026-02-11 10:30:00"}
{"event": "post_created",      "ip": "192.168.1.50", "user_id": "abc-123", "post_id": "xyz-789", "timestamp": "2026-02-11 10:30:00"}
{"event": "request_finished",  "ip": "192.168.1.50", "user_id": "abc-123", "status_code": 201, "duration_ms": 45.2, "timestamp": "2026-02-11 10:30:00"}
```
