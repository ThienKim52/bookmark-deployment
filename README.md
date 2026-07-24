# Deployment & Infrastructure Guide

This directory contains the Docker Compose configurations and service settings required to deploy the **Bookmark Application** in a containerized environment (e.g., local staging, QA, or production).

## System Architecture

The deployment architecture consists of an **NGINX reverse proxy** acting as the API Gateway/Ingress. It routes incoming client requests to either the frontend web application (Portal) or a load-balanced set of backend API instances (**Bookmark Service**), which persist data in **Redis**.

![nginx flow](./nginx.drawio.png)

## Service Descriptions

1. **NGINX Reverse Proxy (`nginx`)**:
   - Acts as the single entry point for all users.
   - Listens on host port `80`.
   - Distributes incoming traffic based on request URI prefixes.
   
2. **Portal (`portal`)**:
   - The user-facing web interface.
   - Served by `ebvn/bookmark-app-portal:dev`.
   - Accessible via the root path `/`.

3. **Bookmark Service (`bookmark_service`)**:
   - The backend REST API written in Go, served by `kimthanthien/bookmark:dev`.
   - Processes link shortening and redirection requests.
   - Resolves database state against Redis.

4. **Redis Cache (`redis`)**:
   - Active in-memory key-value database used by the Bookmark Service to map shortened codes to URLs.

---

## Routing & Port Mapping

| Ingress Path | Target Service | Internal Container Port | Description |
| :--- | :--- | :--- | :--- |
| `http://localhost/` | `portal` | `3000` | Serves the frontend application |
| `http://localhost/api/bookmark_service/` | `bookmark_service` | `8080` | Backend API routes (NGINX strips prefix and forwards) |

### NGINX Path Rewrite Rules
In `nginx/nginx.conf`, the trailing slashes are crucial to forward paths correctly to the backend service:
```nginx
location /api/bookmark_service/ {
    proxy_pass http://bookmark_service/;
}
```
* **Example**: A request to `http://localhost/api/bookmark_service/health-check` is rewritten and forwarded internally to `http://bookmark-service:8080/health-check`.

---

## Environment Configuration

### Bookmark Service Configurations (`./bookmark-service/.env`)
The service reads environment variables via `envconfig`. The default production-ready configurations are:

```env
ADDRESS=redis:6379              # Redis connection address (uses Docker DNS)
BASE_PATH=/api/bookmark_service # Base path for generating Swagger documentation
```

Additional optional environment overrides:
* `SERVICE_NAME`: Custom service identifier (defaults to `health-check-service`).
* `INSTANCE_ID`: Custom unique instance tag. If left blank, a random UUID is generated on startup.
* `LOG_LEVEL`: Logging verbosity level (e.g., `info`, `debug`, `warn`).
* `APP_PORT`: Internal container port for the API server (defaults to `8080`).

---

## Quick Start

### 1. Prerequisites
Ensure you have Docker and Docker Compose (v2 or newer) installed.

### 2. Startup Services
Navigate to this directory and bring up all containers in detached mode:
```bash
docker compose up -d
```

### 3. Verification VM Deployment

The application is deployed on a production VM with the IP address `103.75.183.118`. The routing configuration matches the local environment, mapped via the NGINX reverse proxy on port `80`.

### VM Verification Endpoints

You can verify the live deployment using the following endpoints:
* **Frontend UI**: [http://103.75.183.118/](http://103.75.183.118/)
* **Health Check Endpoint**: [http://103.75.183.118/api/bookmark_service/health-check](http://103.75.183.118/api/bookmark_service/health-check)
* **Swagger API Documentation**: [http://103.75.183.118/api/bookmark_service/swagger/index.html](http://103.75.183.118/api/bookmark_service/swagger/index.html)
* **Swagger Health Check Detail**: [http://103.75.183.118/api/bookmark_service/swagger/index.html#/health-check/get_health_check](http://103.75.183.118/api/bookmark_service/swagger/index.html#/health-check/get_health_check)


### 4. Stop Services
To stop and remove containers and network settings:
```bash
docker compose down
```