# Container Orchestration & Docker Runners

This guide provides technical reference patterns for running Newman inside Docker containers and orchestrating test runs across local and CI environments.

---

## 1. The Official `postman/newman` Images

Postman maintains official Docker images on Docker Hub under the `postman/newman` repository:
- **`postman/newman:alpine`** (Recommended): Minimal, attack-surface-reduced image based on Alpine Linux (~120MB). Fast pull times in CI.
- **`postman/newman:ubuntu`**: Ubuntu-based image containing standard glibc and Ubuntu system packages.

### Why Containerized Newman?
- Eliminates Node.js and npm version discrepancies across developer machines.
- Provides consistent runtime behavior between macOS, Linux, and Windows.
- Runs seamlessly on bare-metal CI agents and Kubernetes build pods without pre-installing npm packages.

---

## 2. Docker Run Command Architecture

The standard containerized execution command:

```bash
docker run --rm -i \
  -v "$(pwd)/newman:/etc/newman" \
  postman/newman:alpine run "/etc/newman/collections/service.postman_collection.json" \
  -e "/etc/newman/environments/local.postman_environment.json" \
  --reporters cli
```

### Breakdown of Flags:
- `--rm`: Automatically removes the container after execution, preventing container accumulation on disk.
- `-i`: Keeps STDIN open for interactive signal handling and log streaming.
- `-v "$(pwd)/newman:/etc/newman"`: Mounts the local `newman/` directory to the container's default `/etc/newman` working path.
- `postman/newman:alpine`: Pinned, lightweight official image.
- `run "..."`: Newman collection runner arguments executed inside the container.

---

## 3. Host Networking Resolution Matrix

Connecting from inside the container to a service running on the host machine depends on the host operating system:

| Host OS | Networking Flag | Base URL Configuration | Notes |
| :--- | :--- | :--- | :--- |
| **Linux** | `--network host` | `http://localhost:8080` | Container shares host network namespace directly. |
| **macOS (Docker Desktop)** | Bridge (default) or `--network host` | `http://host.docker.internal:8080` | Virtual machine barrier prevents direct `localhost` binding; `host.docker.internal` routes to Mac host. |
| **Windows (Docker Desktop)** | Bridge (default) or `--network host` | `http://host.docker.internal:8080` | Same as macOS; requires `host.docker.internal`. |
| **Docker Compose** | Named network (`--network my_network`) | `http://service-name:8080` | Connects directly via Docker internal DNS. |

### Handling macOS/Windows in Scripts:
```bash
case "$(uname -s)" in
  Darwin*|MINGW*|MSYS*|CYGWIN*)
    # macOS & Windows Docker Desktop: bridge networking with host.docker.internal
    NET_FLAG=""
    URL_OVERRIDE="--env-var baseUrl=http://host.docker.internal:8080"
    ;;
  *)
    # Linux: native host networking
    NET_FLAG="--network host"
    URL_OVERRIDE=""
    ;;
esac

docker run --rm -i \
  ${NET_FLAG} \
  -v "$(pwd)/newman:/etc/newman" \
  postman/newman:alpine run "/etc/newman/collections/service.postman_collection.json" \
  -e "/etc/newman/environments/local.postman_environment.json" \
  ${URL_OVERRIDE} \
  --reporters cli
```

---

## 4. Docker Compose Orchestration

For end-to-end integration testing in local development or ephemeral staging environments, Newman can be declared as a service inside `compose.yaml`:

```yaml
name: service-e2e-suite

services:
  app:
    build: .
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 5s
      timeout: 3s
      retries: 5

  newman-runner:
    image: postman/newman:alpine
    depends_on:
      app:
        condition: service_healthy
    volumes:
      - ./newman:/etc/newman
    command: >
      run /etc/newman/collections/service.postman_collection.json
      -e /etc/newman/environments/local.postman_environment.json
      --env-var baseUrl=http://app:8080
      --reporters cli,junit
      --reporter-junit-export /etc/newman/reports/results.xml
```

### Key Practices:
1. **Healthcheck Choreography**: The `newman-runner` service MUST use `depends_on` with `condition: service_healthy` so it never runs before the backend application is fully booted and accepting HTTP traffic.
2. **Internal Service DNS**: Configure `--env-var baseUrl=http://app:8080` to target the application service directly over Docker's internal DNS network.
