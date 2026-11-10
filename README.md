# Conduit – Dockerized Fullstack Application

Docker Compose setup for the Conduit application, using Angular 17, Django 1.10.5 and PostgreSQL 14.

The application consists of three services: an Nginx frontend, a Django backend running with Gunicorn, and a PostgreSQL database.

## Table of Contents

- [Project Objectives](#project-objectives)
- [Quickstart](#quickstart)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Architecture](#architecture)
- [Usage](#usage)
  - [Configuration](#configuration)
  - [Container Management](#container-management)
  - [Django Management](#django-management)
  - [Logs and Monitoring](#logs-and-monitoring)
- [Persistence and Recovery](#persistence-and-recovery)
- [Verification](#verification)
- [Limitations](#limitations)

## Project Objectives

- Containerize the existing Angular and Django applications using multi-stage Docker builds.
- Replace SQLite with PostgreSQL.
- Serve the Angular frontend through Nginx and the Django backend through Gunicorn.
- Connect all services using Docker Compose.
- Manage configuration through environment variables.
- Persist application data independently of container lifecycles.
- Configure automatic container restarts.
- Deploy and verify the application on an Ubuntu server.

The original application sources were imported in a separate commit to keep the containerization changes traceable.

## Quickstart

### 1. Clone the repository

```bash
git clone -b develop https://github.com/YOUR-USERNAME/conduit-container.git
cd conduit-container
```

Replace `YOUR-USERNAME` with the repository owner's GitHub username.

### 2. Create the environment files

```bash
cp .env.example .env
cp conduit-backend/.env.example conduit-backend/.env
```

Configure the PostgreSQL credentials and Django settings as described in [Configuration](#configuration).

### 3. Build and start

```bash
docker compose up -d --build
```

### 4. Initialize Django

Apply database migrations:

```bash
docker compose exec backend python manage.py migrate
```

Collect static files:

```bash
docker compose exec backend python manage.py collectstatic --noinput
```

Create an admin account if needed:

```bash
docker compose exec backend python manage.py createsuperuser
```

### 5. Access the application

| Application | URL |
|---|---|
| Frontend | `http://localhost:8282` |
| Django Admin | `http://localhost:8282/admin/` |
| API example | `http://localhost:8282/api/tags` |

For remote access, replace `localhost` with the server's IP address. The frontend port defaults to `8282` and can be changed through the root `.env` file.

## Prerequisites

- Docker Engine or Docker Desktop
- Docker Compose v2
- Git

Python, Node.js, PostgreSQL and Nginx are provided by the Docker images and do not need to be installed on the host.

## Repository Structure

```text
conduit-container/
├── .github/
│   └── workflows/
│       └── check-open-pr.yaml
├── conduit-backend/
│   ├── conduit/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── .env.example
│   └── requirements.txt
├── conduit-frontend/
│   ├── src/
│   ├── nginx/
│   │   └── default.conf
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── angular.json
│   └── package.json
├── compose.yml
├── .env.example
├── .gitignore
└── README.md
```

The root `compose.yml` defines the services, network communication, volumes and restart policies. Frontend and backend have separate Dockerfiles and build contexts.

## Architecture

| Service | Stack | Internal Port | Host Access |
|---|---|---|---|
| `frontend` | Angular 17, Nginx | 80 | 8282 (default) |
| `backend` | Python 3.5.2, Django 1.10.5, Gunicorn 19.9.0 | 8000 | Not published |
| `database` | PostgreSQL 14 | 5432 | Not published |

Request flow:

```text
Browser
   |
   | HTTP :8282
   v
Frontend (Nginx :80)
   |
   | /api/ and /admin/
   v
Backend (Gunicorn :8000)
   |
   | PostgreSQL :5432
   v
Database
   |
   v
Persistent Docker Volume
```

The frontend is built with Node.js 20 and served by Nginx. Nginx forwards API and admin requests to `backend:8000`, serves collected Django files under `/static/`, and handles Angular routes through an `index.html` fallback.

Gunicorn runs the Django WSGI application on port `8000`. The backend connects to PostgreSQL using the Docker Compose service name `database`.

Only the frontend port is published on the host. Backend and database communication stays within the Compose network.

## Usage

### Configuration

Two environment files are required.

#### Root `.env`

Contains Docker Compose and PostgreSQL settings.

```dotenv
POSTGRES_DB=conduit
POSTGRES_USER=conduit
POSTGRES_PASSWORD=CHANGE_ME

FRONTEND_PORT=8282
```

| Variable | Description |
|---|---|
| `POSTGRES_DB` | Database name |
| `POSTGRES_USER` | Database user |
| `POSTGRES_PASSWORD` | Database password |
| `FRONTEND_PORT` | Published frontend port (default: 8282) |

The frontend port uses a default value in `compose.yml`:

```yaml
ports:
  - "${FRONTEND_PORT:-8282}:80"
```

The backend port is fixed and exposed only within the Docker network:

```yaml
expose:
  - "8000"
```

No host port mapping is required for Django or PostgreSQL.

#### Backend `.env`

Contains Django application settings.

```dotenv
DJANGO_SECRET_KEY=CHANGE_ME
DJANGO_DEBUG=False
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1,YOUR_SERVER_IP
DJANGO_STATIC_ROOT=/app/staticfiles
```

Generate a random secret key:

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(48))"
```

`DJANGO_ALLOWED_HOSTS` accepts comma-separated hostnames or IP addresses without a protocol or port number.

Replace `YOUR_SERVER_IP` with the server's actual IP address for remote deployment. Remove the placeholder if the application is only used locally.

The real `.env` files are excluded from Git. Only `.env.example` files should be committed.

### Container Management

Build and start:

```bash
docker compose up -d --build
```

Check service status:

```bash
docker compose ps
```

Restart a service:

```bash
docker compose restart backend
```

Stop and remove containers:

```bash
docker compose down
```

This preserves named Docker volumes. Running `docker compose down -v` also removes volumes and can delete stored database data.

### Django Management

Apply migrations:

```bash
docker compose exec backend python manage.py migrate
```

Check migration status:

```bash
docker compose exec backend python manage.py showmigrations
```

Collect static files:

```bash
docker compose exec backend python manage.py collectstatic --noinput
```

Create a superuser:

```bash
docker compose exec backend python manage.py createsuperuser
```

Open a Django shell:

```bash
docker compose exec backend python manage.py shell
```

### Logs and Monitoring

View all service logs:

```bash
docker compose logs
```

Follow backend logs:

```bash
docker compose logs -f backend
```

View the last 50 database log entries:

```bash
docker compose logs --tail=50 database
```

Check the backend restart policy and counter:

```bash
docker inspect \
  -f 'Policy={{.HostConfig.RestartPolicy.Name}} RestartCount={{.RestartCount}}' \
  "$(docker compose ps -q backend)"
```

### **Exporting Logs**

Container logs can be exported to a file for later inspection.

Save backend logs:

```sh
docker compose logs --no-color backend > backend-logs.txt
```

Save logs from all services:

```sh
docker compose logs --no-color > conduit-logs.txt
```

Append new log output to an existing file:

```sh
docker compose logs --no-color backend >> backend-logs.txt
```

Exported logs may contain sensitive information and should not be committed to the repository.

## Persistence and Recovery

### Data Persistence

PostgreSQL stores its database files in a named Docker volume. Django's collected static files are stored in a separate shared volume used by Nginx.

Named volumes are preserved when containers are removed with `docker compose down`.

To verify database persistence:

1. Create a user or article in Conduit.
2. Run `docker compose down`.
3. Run `docker compose up -d`.
4. Check that the previously created data still exists.

The test was performed on the Ubuntu VM. The Django superuser remained available after the containers were recreated.

### Automatic Recovery

The services use the following restart policy:

```yaml
restart: unless-stopped
```

To test backend recovery, terminate the Gunicorn master process:

```bash
docker compose exec -T backend python -c \
  "import os, signal; os.kill(1, signal.SIGTERM)"
```

Check whether Docker restarted the container:

```bash
docker compose ps

docker inspect \
  -f 'Status={{.State.Status}} RestartCount={{.RestartCount}}' \
  "$(docker compose ps -q backend)"
```

The backend automatically restarted during testing on the Ubuntu VM.

## Verification

The application was tested locally and on an Ubuntu server.

| Test | Result |
|---|---|
| Docker image builds and service startup | Passed |
| PostgreSQL migrations | Passed |
| Angular frontend and CSS | Passed |
| Nginx reverse proxy and API (`/api/tags`) | HTTP 200 |
| Django Admin login | Passed |
| User registration | Passed |
| Login and logout | Passed |
| Article creation and editing | Passed |
| PostgreSQL persistence after container recreation | Passed |
| Automatic backend restart | Passed |

The VM deployment was accessed through the server's public IP address on port `8282`.

## Limitations

The backend uses Python 3.5.2, Django 1.10.5 and Gunicorn 19.9.0 to meet the project requirements. These versions are no longer maintained.

The current VM deployment uses HTTP without TLS encryption.

This setup is intended for development and course evaluation, not production use. A production deployment would require supported dependencies, HTTPS, stricter access controls and further security review.

Real credentials are kept outside version control, and the database and backend ports are not published to the host.

Known issue: The original Angular editor creates a new article when saving changes to an existing article. This behavior is inherited from the upstream application and is outside the scope of this containerization project. 