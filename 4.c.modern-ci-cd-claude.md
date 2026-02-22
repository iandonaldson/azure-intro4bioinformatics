# Deploying multi-container apps on Azure: a modern guide

**This tutorial walks you through building, containerizing, and deploying a two-container web application to Azure using four modern practices: multi-stage Docker builds, Docker Compose orchestration, GitHub Container Registry for image storage, and Buildkite for CI/CD.** It modernizes the foundational approach from the [azure-intro4bioinformatics tutorial](https://github.com/iandonaldson/azure-intro4bioinformatics/blob/main/4.b.practical_container_deployment_on_azure.md), which introduced container deployment on Azure using manual Docker builds and Azure Container Registry. Here, we replace that workflow with a **GitHub-first development approach** — coding in GitHub Codespaces, storing images in GHCR, and automating deployments through Buildkite pipelines to Azure Container Apps.

By the end, you will have a fully automated pipeline: push code → Buildkite builds optimized images → pushes to GHCR → deploys to Azure Container Apps. No manual Docker commands. No copying credentials around. No "works on my machine."

---

## What we are building and why it matters

Our application has two containers working together:

- **Frontend (`web`)**: A Flask application serving HTML pages and proxying API requests. It presents a simple task-tracker interface.
- **Backend (`api`)**: A FastAPI service with a SQLite database that handles CRUD operations for tasks. It stores data in a persistent volume.

The frontend talks to the backend over an internal network. Only the frontend is exposed to the internet. This is a deliberately simple architecture that mirrors real-world patterns — a web tier and a data tier — without the complexity of a full database server.

### The four modern concepts

The original tutorial likely used a straightforward approach: build a Docker image locally, push it to Azure Container Registry (ACR), and deploy to Azure Container Instances (ACI) via the Azure CLI. That works for learning, but production teams need more. Here is what we are adding and why:

1. **Multi-stage Dockerfiles** reduce image sizes by **70% or more**, exclude build tools and their CVEs from production, and enable a single Dockerfile to serve development, testing, and production.
2. **Docker Compose** orchestrates multiple containers with one command, manages networking and volumes, and keeps your development environment identical to production.
3. **GitHub Container Registry (GHCR)** keeps your images next to your code on GitHub, offers free storage for public packages, and integrates natively with GitHub's authentication.
4. **Buildkite** provides a hybrid CI/CD architecture where your code and secrets never leave your infrastructure, scales to 100,000+ concurrent agents, and uses per-seat pricing instead of per-minute billing.

### Prerequisites

You need these accounts and tools before starting:

- A [GitHub account](https://github.com) with Codespaces access (free tier includes 120 core-hours/month)
- An [Azure account](https://azure.microsoft.com/free/) with an active subscription
- A [Buildkite account](https://buildkite.com/signup) (free tier available for small teams)
- Basic familiarity with Git, Docker, and command-line tools

---

## Step 1: Create the GitHub repository and project structure

Start by creating a new repository on GitHub. This repository will hold everything: application code, Dockerfiles, Compose configuration, Codespaces setup, and the Buildkite pipeline.

Go to [github.com/new](https://github.com/new) and create a repository named `modern-azure-deploy`. Initialize it with a README and a Python `.gitignore`. Clone it locally or — better yet — open it directly in Codespaces (we will configure that next).

Create this directory structure:

```
modern-azure-deploy/
├── .devcontainer/
│   └── devcontainer.json
├── .buildkite/
│   └── pipeline.yml
├── web/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── app.py
│   └── templates/
│       ├── base.html
│       ├── index.html
│       └── tasks.html
├── api/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── main.py
├── compose.yaml
├── compose.override.yaml
├── .dockerignore
├── .env.example
└── README.md
```

**Why this structure matters:** Separating `web/` and `api/` into their own directories gives each service its own Dockerfile and dependency file. The `.devcontainer/` folder configures Codespaces. The `.buildkite/` folder holds the CI/CD pipeline. The `compose.yaml` at the root ties everything together.

---

## Step 2: Develop in the cloud with GitHub Codespaces

GitHub Codespaces gives every developer an identical, pre-configured environment in the cloud. No more "install Python 3.12 and Docker on your laptop" instructions that break differently on every operating system. When someone opens your repo in Codespaces, they get a full VS Code editor connected to a Linux container with all tools pre-installed.

### Configure devcontainer.json

Create `.devcontainer/devcontainer.json`:

```json
{
  "name": "Modern Azure Deploy",
  "image": "mcr.microsoft.com/devcontainers/python:1-3.12-bullseye",

  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": {},
    "ghcr.io/devcontainers/features/azure-cli:1": {},
    "ghcr.io/devcontainers/features/github-cli:1": {}
  },

  "customizations": {
    "vscode": {
      "extensions": [
        "ms-python.python",
        "ms-python.vscode-pylance",
        "ms-azuretools.vscode-docker",
        "ms-azuretools.vscode-azurecontainerapps",
        "redhat.vscode-yaml"
      ],
      "settings": {
        "python.defaultInterpreterPath": "/usr/local/bin/python",
        "editor.formatOnSave": true,
        "editor.rulers": [100]
      }
    }
  },

  "forwardPorts": [5000, 8000],

  "portsAttributes": {
    "5000": { "label": "Frontend (Flask)", "onAutoForward": "notify" },
    "8000": { "label": "Backend (FastAPI)", "onAutoForward": "silent" }
  },

  "containerEnv": {
    "PYTHONDONTWRITEBYTECODE": "1",
    "PYTHONUNBUFFERED": "1"
  },

  "postCreateCommand": "pip install -r web/requirements.txt && pip install -r api/requirements.txt",
  "postStartCommand": "docker compose up -d"
}
```

**What each section does:**

- **`image`** uses Microsoft's official Python dev container, which includes Git, pip, and common tools. You don't need a custom Dockerfile for the development environment itself.
- **`features`** installs Docker-in-Docker (so you can run `docker compose` inside Codespaces), the Azure CLI (for deployment commands), and the GitHub CLI (for GHCR authentication).
- **`forwardPorts`** automatically makes ports 5000 and 8000 accessible in your browser when services start.
- **`postCreateCommand`** runs once when the Codespace is first created, installing Python dependencies for both services.
- **`postStartCommand`** runs every time the Codespace starts, bringing up both containers with Docker Compose.

> **Common pitfall:** Codespaces environments are ephemeral outside of `/workspaces/`. If you need persistent data across rebuilds, store it in the workspace directory. The `docker-in-docker` feature is required to run Docker commands — without it, `docker` and `docker compose` will not be available.

To open your repository in Codespaces, click the green **Code** button on your GitHub repository page, select the **Codespaces** tab, and click **Create codespace on main**. After a minute or two of setup, you will have a fully configured development environment.

For more on configuration options, see the [Dev Container specification reference](https://containers.dev/implementors/json_reference/) and [GitHub Codespaces documentation](https://docs.github.com/en/codespaces).

---

## Step 3: Build the application

### The backend API service

Create `api/requirements.txt`:

```
fastapi==0.115.6
uvicorn[standard]==0.34.0
```

Create `api/main.py`:

```python
"""Task API — lightweight CRUD service backed by SQLite."""
import sqlite3
import os
from contextlib import contextmanager
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel

app = FastAPI(title="Task API", version="1.0.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

DB_PATH = os.getenv("DB_PATH", "/data/tasks.db")


@contextmanager
def get_db():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    try:
        yield conn
    finally:
        conn.close()


def init_db():
    os.makedirs(os.path.dirname(DB_PATH), exist_ok=True)
    with get_db() as conn:
        conn.execute("""
            CREATE TABLE IF NOT EXISTS tasks (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                title TEXT NOT NULL,
                description TEXT DEFAULT '',
                completed BOOLEAN DEFAULT 0,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        conn.commit()


class TaskCreate(BaseModel):
    title: str
    description: str = ""


class TaskUpdate(BaseModel):
    title: str | None = None
    description: str | None = None
    completed: bool | None = None


@app.on_event("startup")
def startup():
    init_db()


@app.get("/health")
def health():
    try:
        with get_db() as conn:
            conn.execute("SELECT 1")
        return {"status": "healthy"}
    except Exception as e:
        raise HTTPException(status_code=503, detail=str(e))


@app.get("/api/tasks")
def list_tasks():
    with get_db() as conn:
        rows = conn.execute("SELECT * FROM tasks ORDER BY created_at DESC").fetchall()
        return [dict(row) for row in rows]


@app.post("/api/tasks", status_code=201)
def create_task(task: TaskCreate):
    with get_db() as conn:
        cursor = conn.execute(
            "INSERT INTO tasks (title, description) VALUES (?, ?)",
            (task.title, task.description),
        )
        conn.commit()
        row = conn.execute("SELECT * FROM tasks WHERE id = ?", (cursor.lastrowid,)).fetchone()
        return dict(row)


@app.put("/api/tasks/{task_id}")
def update_task(task_id: int, task: TaskUpdate):
    with get_db() as conn:
        existing = conn.execute("SELECT * FROM tasks WHERE id = ?", (task_id,)).fetchone()
        if not existing:
            raise HTTPException(status_code=404, detail="Task not found")
        updates = {k: v for k, v in task.model_dump().items() if v is not None}
        if updates:
            set_clause = ", ".join(f"{k} = ?" for k in updates)
            conn.execute(f"UPDATE tasks SET {set_clause} WHERE id = ?",
                         [*updates.values(), task_id])
            conn.commit()
        row = conn.execute("SELECT * FROM tasks WHERE id = ?", (task_id,)).fetchone()
        return dict(row)


@app.delete("/api/tasks/{task_id}", status_code=204)
def delete_task(task_id: int):
    with get_db() as conn:
        result = conn.execute("DELETE FROM tasks WHERE id = ?", (task_id,))
        conn.commit()
        if result.rowcount == 0:
            raise HTTPException(status_code=404, detail="Task not found")
```

This is a straightforward CRUD API. **SQLite is embedded in Python** — no external database server needed. The database file lives at `/data/tasks.db`, which we will mount as a Docker volume for persistence.

### The frontend web service

Create `web/requirements.txt`:

```
flask==3.1.0
requests==2.32.3
gunicorn==23.0.0
```

Create `web/app.py`:

```python
"""Flask frontend — renders HTML and proxies API calls to the backend."""
import os
import requests
from flask import Flask, render_template, request, redirect, url_for

app = Flask(__name__)

API_URL = os.getenv("API_URL", "http://api:8000")


@app.route("/")
def index():
    return render_template("index.html")


@app.route("/tasks")
def tasks():
    try:
        response = requests.get(f"{API_URL}/api/tasks", timeout=5)
        task_list = response.json()
    except requests.exceptions.ConnectionError:
        task_list = []
    return render_template("tasks.html", tasks=task_list)


@app.route("/tasks/add", methods=["POST"])
def add_task():
    title = request.form.get("title", "").strip()
    description = request.form.get("description", "").strip()
    if title:
        requests.post(f"{API_URL}/api/tasks",
                       json={"title": title, "description": description}, timeout=5)
    return redirect(url_for("tasks"))


@app.route("/tasks/<int:task_id>/toggle", methods=["POST"])
def toggle_task(task_id):
    resp = requests.get(f"{API_URL}/api/tasks", timeout=5)
    current = next((t for t in resp.json() if t["id"] == task_id), None)
    if current:
        requests.put(f"{API_URL}/api/tasks/{task_id}",
                      json={"completed": not current["completed"]}, timeout=5)
    return redirect(url_for("tasks"))


@app.route("/tasks/<int:task_id>/delete", methods=["POST"])
def delete_task(task_id):
    requests.delete(f"{API_URL}/api/tasks/{task_id}", timeout=5)
    return redirect(url_for("tasks"))


@app.route("/health")
def health():
    return {"status": "healthy"}, 200


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```

Create `web/templates/base.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Task Tracker{% endblock %}</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
               max-width: 800px; margin: 0 auto; padding: 20px; background: #f5f5f5; }
        nav { background: #2563eb; padding: 15px 20px; border-radius: 8px; margin-bottom: 20px; }
        nav a { color: white; text-decoration: none; margin-right: 20px; font-weight: 500; }
        nav a:hover { text-decoration: underline; }
        .container { background: white; padding: 25px; border-radius: 8px; box-shadow: 0 1px 3px rgba(0,0,0,0.1); }
        h1 { color: #1e293b; margin-bottom: 15px; }
        .btn { padding: 8px 16px; border: none; border-radius: 6px; cursor: pointer; font-size: 14px; }
        .btn-primary { background: #2563eb; color: white; }
        .btn-danger { background: #dc2626; color: white; }
        .btn-success { background: #16a34a; color: white; }
        input, textarea { padding: 8px 12px; border: 1px solid #d1d5db; border-radius: 6px;
                          font-size: 14px; width: 100%; margin-bottom: 10px; }
        .task { display: flex; align-items: center; padding: 12px; border-bottom: 1px solid #e5e7eb; }
        .task.completed span { text-decoration: line-through; color: #9ca3af; }
        .task span { flex: 1; }
        .task form { margin-left: 8px; }
    </style>
</head>
<body>
    <nav>
        <a href="/">Home</a>
        <a href="/tasks">Tasks</a>
    </nav>
    <div class="container">
        {% block content %}{% endblock %}
    </div>
</body>
</html>
```

Create `web/templates/index.html`:

```html
{% extends "base.html" %}
{% block content %}
<h1>Welcome to Task Tracker</h1>
<p style="margin-bottom: 15px; color: #475569;">
    A modern multi-container application deployed on Azure Container Apps.
</p>
<p style="color: #64748b;">
    <strong>Architecture:</strong> Flask frontend → FastAPI backend → SQLite database.
    Built with multi-stage Docker builds, orchestrated with Docker Compose,
    images stored on GHCR, deployed via Buildkite CI/CD.
</p>
<a href="/tasks" class="btn btn-primary" style="display:inline-block; margin-top:15px;">View Tasks →</a>
{% endblock %}
```

Create `web/templates/tasks.html`:

```html
{% extends "base.html" %}
{% block title %}Tasks{% endblock %}
{% block content %}
<h1>Tasks</h1>
<form action="/tasks/add" method="post" style="margin: 15px 0; display:flex; gap:10px;">
    <input type="text" name="title" placeholder="Task title" required style="flex:2;">
    <input type="text" name="description" placeholder="Description (optional)" style="flex:3;">
    <button type="submit" class="btn btn-primary">Add</button>
</form>
{% for task in tasks %}
<div class="task {% if task.completed %}completed{% endif %}">
    <span><strong>{{ task.title }}</strong> — {{ task.description }}</span>
    <form action="/tasks/{{ task.id }}/toggle" method="post">
        <button class="btn btn-success" type="submit">
            {{ "Undo" if task.completed else "Done" }}
        </button>
    </form>
    <form action="/tasks/{{ task.id }}/delete" method="post">
        <button class="btn btn-danger" type="submit">Delete</button>
    </form>
</div>
{% else %}
<p style="color: #9ca3af; margin-top: 15px;">No tasks yet. Add one above!</p>
{% endfor %}
{% endblock %}
```

---

## Step 4: Multi-stage Dockerfiles that do more with less

A multi-stage Dockerfile uses multiple `FROM` statements. Each `FROM` begins a new "stage" with a fresh filesystem. You copy only what you need between stages using `COPY --from=<stage>`. The result: production images that contain your application and its runtime dependencies — nothing else. No compilers. No package managers. No build artifacts. **Typical size reductions exceed 70%**, and the smaller attack surface significantly improves security.

For a deeper dive, see the [official multi-stage build documentation](https://docs.docker.com/build/building/multi-stage/).

### Backend Dockerfile (api/Dockerfile)

```dockerfile
# ============================================
# Stage 1: Install dependencies in a venv
# ============================================
FROM python:3.12-slim AS builder

# Install build tools needed for compiling Python packages
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential gcc \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Create an isolated virtual environment
RUN python -m venv /venv
ENV PATH="/venv/bin:$PATH"

# Copy and install dependencies FIRST (cached unless requirements.txt changes)
COPY requirements.txt .
RUN pip install --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# ============================================
# Stage 2: Production — clean, minimal image
# ============================================
FROM python:3.12-slim AS production

# OCI labels link this image to the GitHub repository
LABEL org.opencontainers.image.source="https://github.com/YOUR_GITHUB_USERNAME/modern-azure-deploy"
LABEL org.opencontainers.image.description="Task Tracker API service"

# Create a non-root user for security
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Copy ONLY the virtual environment from the builder — no gcc, no build-essential
COPY --from=builder /venv /venv
ENV PATH="/venv/bin:$PATH"

WORKDIR /app
COPY main.py .

# Create data directory for SQLite and set ownership
RUN mkdir -p /data && chown -R appuser:appuser /app /data
USER appuser

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Frontend Dockerfile (web/Dockerfile)

```dockerfile
# ============================================
# Stage 1: Install dependencies
# ============================================
FROM python:3.12-slim AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential gcc \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
RUN python -m venv /venv
ENV PATH="/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# ============================================
# Stage 2: Production
# ============================================
FROM python:3.12-slim AS production

LABEL org.opencontainers.image.source="https://github.com/YOUR_GITHUB_USERNAME/modern-azure-deploy"
LABEL org.opencontainers.image.description="Task Tracker web frontend"

RUN groupadd -r appuser && useradd -r -g appuser appuser

COPY --from=builder /venv /venv
ENV PATH="/venv/bin:$PATH"

WORKDIR /app
COPY app.py .
COPY templates/ templates/

RUN chown -R appuser:appuser /app
USER appuser

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:5000/health')"

EXPOSE 5000
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "--timeout", "120", "app:app"]
```

### Why this pattern works

The key insight is the **layer caching strategy**. By copying `requirements.txt` before the application code, Docker caches the expensive `pip install` step. Code changes — which happen constantly — only invalidate the final `COPY` layers. Dependency installation only reruns when `requirements.txt` changes.

The production stage starts from a fresh `python:3.12-slim` base. It copies in the virtual environment from the builder, which contains all installed packages in a clean, portable directory. **The builder's `gcc`, `build-essential`, and `apt` caches are completely excluded** from the production image.

Running as a non-root user (`appuser`) is a security best practice that prevents container escape attacks from gaining root access on the host.

### Building specific stages

```bash
# Build only the production stage (default — last stage)
docker build -t myapp-api:prod api/

# Build the builder stage for debugging dependency issues
docker build --target builder -t myapp-api:builder api/

# Compare image sizes to see the difference
docker images | grep myapp-api
```

Expected output:
```
myapp-api    prod      a1b2c3d4e5f6   10 seconds ago   182MB
myapp-api    builder   f6e5d4c3b2a1   15 seconds ago   487MB
```

Create a `.dockerignore` in the project root to prevent unnecessary files from entering the build context:

```
.git
__pycache__
*.pyc
.venv
.env
*.md
.devcontainer/
.buildkite/
compose*.yaml
```

> **Common pitfalls:** Placing `COPY . .` before `RUN pip install` invalidates the pip cache on every code change. Forgetting `--no-cache-dir` on pip leaves cache artifacts in the image layer. Using `python:3.12` instead of `python:3.12-slim` adds **hundreds of megabytes** of unnecessary packages.

---

## Step 5: Docker Compose orchestrates the full stack

Docker Compose defines your multi-container application in a single YAML file. Instead of running separate `docker build` and `docker run` commands with complex flags, you describe your entire stack declaratively and bring it up with one command.

Docker Compose v2 is the current version — a Go rewrite of the original Python-based tool. Use `docker compose` (space, not hyphen). The old `docker-compose` command is deprecated. The `version:` key in compose files is no longer required and is ignored by v2.

For full specification details, see the [Docker Compose documentation](https://docs.docker.com/compose/).

### Production configuration (compose.yaml)

```yaml
services:
  web:
    build:
      context: ./web
      dockerfile: Dockerfile
      target: production
    image: ghcr.io/YOUR_GITHUB_USERNAME/modern-azure-deploy-web:latest
    ports:
      - "5000:5000"
    environment:
      - API_URL=http://api:8000
    depends_on:
      api:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:5000/health')"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    restart: unless-stopped
    networks:
      - app-network

  api:
    build:
      context: ./api
      dockerfile: Dockerfile
      target: production
    image: ghcr.io/YOUR_GITHUB_USERNAME/modern-azure-deploy-api:latest
    ports:
      - "8000:8000"
    environment:
      - DB_PATH=/data/tasks.db
    volumes:
      - api-data:/data
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 5s
    restart: unless-stopped
    networks:
      - app-network

volumes:
  api-data:

networks:
  app-network:
    driver: bridge
```

### Development override (compose.override.yaml)

Docker Compose automatically merges `compose.override.yaml` on top of `compose.yaml` when you run `docker compose up`. This file overrides production settings with development-friendly defaults.

```yaml
services:
  web:
    build:
      target: production
    volumes:
      - ./web:/app
    environment:
      - FLASK_DEBUG=1
    command: python app.py
    ports:
      - "5000:5000"

  api:
    build:
      target: production
    volumes:
      - ./api:/app
      - api-data:/data
    command: uvicorn main:app --host 0.0.0.0 --port 8000 --reload
    ports:
      - "8000:8000"
```

**What the override changes for development:**

- **Bind mounts** (`volumes: - ./web:/app`) map your local source code into the container. Code changes appear instantly without rebuilding.
- **`--reload` flag** on uvicorn and Flask's debug mode enable hot-reload — save a file, the server restarts automatically.
- **`command` overrides** replace gunicorn (production WSGI server) with Flask's development server and uvicorn with reload enabled.

### Key Compose concepts explained

**`depends_on` with `condition: service_healthy`** is critical. Without the health check condition, Compose only waits for the container to *start* — not for the application inside to be *ready*. The Flask frontend needs the API to be accepting requests before it tries to connect. The health check ensures this.

**Named volumes** (`api-data`) persist data across container restarts and rebuilds. The SQLite database file lives in this volume. Without it, every `docker compose down` would destroy your data.

**Networks** allow containers to find each other by service name. The Flask app connects to `http://api:8000` — Docker's built-in DNS resolves `api` to the correct container IP. You never need to know or hardcode IP addresses.

### Running the application

```bash
# Start everything (dev mode, with overrides applied automatically)
docker compose up --build

# Expected output:
# ✔ Network modern-azure-deploy_app-network  Created
# ✔ Volume "modern-azure-deploy_api-data"    Created
# ✔ Container modern-azure-deploy-api-1      Healthy
# ✔ Container modern-azure-deploy-web-1      Started
# api-1  | INFO:     Uvicorn running on http://0.0.0.0:8000
# web-1  | * Running on http://0.0.0.0:5000

# Start in production mode (skip the override file)
docker compose -f compose.yaml up --build -d

# Check service health
docker compose ps

# View logs
docker compose logs -f web

# Tear down everything
docker compose down

# Tear down AND remove volumes (deletes data)
docker compose down -v
```

> **Common pitfall:** Using `docker compose down -v` deletes named volumes, destroying persistent data. Use `docker compose down` (without `-v`) during normal development.

---

## Step 6: Push images to GitHub Container Registry

GitHub Container Registry (GHCR) stores your Docker images at `ghcr.io`. Unlike Docker Hub, GHCR is free for public images with no pull rate limits, and it lives right next to your code on GitHub. Unlike Azure Container Registry, it requires no Azure subscription.

Full documentation: [Working with the Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry).

### Authenticate to GHCR

You need a **classic** Personal Access Token (PAT). GHCR does not support fine-grained PATs.

1. Go to [github.com/settings/tokens](https://github.com/settings/tokens)
2. Click **Generate new token (classic)**
3. Select scopes: **`write:packages`**, **`read:packages`**, **`delete:packages`**
4. Generate and copy the token

```bash
# Store the token in an environment variable
export CR_PAT=ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Log in to GHCR
echo $CR_PAT | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin

# Expected output:
# Login Succeeded
```

### Build, tag, and push

Image names on GHCR follow the format `ghcr.io/OWNER/IMAGE_NAME:TAG`. **The owner must be lowercase** — `ghcr.io/MyUser/app` will fail; use `ghcr.io/myuser/app`.

```bash
# Build both images
docker compose build

# Tag with version and git SHA for traceability
export GIT_SHA=$(git rev-parse --short HEAD)
export GHCR_PREFIX=ghcr.io/your-github-username

# Tag the web image
docker tag modern-azure-deploy-web:latest ${GHCR_PREFIX}/modern-azure-deploy-web:latest
docker tag modern-azure-deploy-web:latest ${GHCR_PREFIX}/modern-azure-deploy-web:${GIT_SHA}

# Tag the api image
docker tag modern-azure-deploy-api:latest ${GHCR_PREFIX}/modern-azure-deploy-api:latest
docker tag modern-azure-deploy-api:latest ${GHCR_PREFIX}/modern-azure-deploy-api:${GIT_SHA}

# Push all tags
docker push ${GHCR_PREFIX}/modern-azure-deploy-web:latest
docker push ${GHCR_PREFIX}/modern-azure-deploy-web:${GIT_SHA}
docker push ${GHCR_PREFIX}/modern-azure-deploy-api:latest
docker push ${GHCR_PREFIX}/modern-azure-deploy-api:${GIT_SHA}
```

Expected output:
```
The push refers to repository [ghcr.io/your-username/modern-azure-deploy-web]
a1b2c3d4: Pushed
e5f6a7b8: Pushed
latest: digest: sha256:abc123... size: 1234
```

### Make packages visible

By default, GHCR packages are **private**. To make them pullable without authentication:

1. Go to your GitHub profile → **Packages**
2. Click the package name
3. **Package settings** → **Danger Zone** → **Change visibility** → **Public**

For Azure deployments using private images, you will provide registry credentials to Azure (covered in Step 9).

### Link packages to your repository

Add the `org.opencontainers.image.source` label in your Dockerfiles (already included above). This label automatically links the package to your repository on GitHub's UI, making it appear in the repository's sidebar and enabling `GITHUB_TOKEN` permissions.

> **Common pitfall:** Images pushed with a PAT from the CLI are not automatically linked to any repository. Without the OCI source label, you must manually connect the package through GitHub's web interface.

---

## Step 7: Automate everything with Buildkite

### What Buildkite is and how it differs from GitHub Actions

Buildkite is a CI/CD platform with a **hybrid architecture**: Buildkite manages the orchestration (web dashboard, scheduling, logs) as a SaaS, while you run lightweight **agents** on your own infrastructure that execute the actual builds. Your source code and secrets never leave your network.

This is the fundamental architectural difference from GitHub Actions, where both orchestration and execution happen on GitHub's infrastructure (or on self-hosted runners that are harder to manage).

| Aspect | Buildkite | GitHub Actions |
|--------|-----------|----------------|
| **Where builds run** | Your servers, VMs, or containers | GitHub-hosted runners (or self-hosted) |
| **Code exposure** | Stays on your infrastructure | Sent to GitHub runners |
| **Pricing** | Per seat (~$30/user/month), unlimited builds | Per minute after free tier (2,000 min/month) |
| **Scale** | 100,000+ concurrent agents | Concurrent job limits apply |
| **Setup** | More upfront work (agent installation) | Near-zero for hosted runners |
| **Best for** | Teams >50 engineers, security-sensitive orgs | Small-medium teams, simple workflows |

**Why choose Buildkite here?** For production deployments where you handle sensitive data or need full control over the build environment, Buildkite's model is compelling. Your Azure credentials, database passwords, and API keys never touch a third-party server. For open-source projects or small teams, GitHub Actions may be simpler.

Full documentation: [Buildkite Docs](https://buildkite.com/docs).

### Set up a Buildkite agent

The Buildkite agent is an [open-source Go binary](https://github.com/buildkite/agent) that polls Buildkite for jobs, checks out code, runs commands, and reports results. You need at least one agent running somewhere with Docker installed.

**Option A: Run the agent on an Azure VM (recommended for Azure deployments)**

```bash
# Create an Azure VM
az vm create \
  --resource-group my-resource-group \
  --name buildkite-agent \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys

# SSH into the VM
ssh azureuser@<VM_PUBLIC_IP>

# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker azureuser

# Install the Buildkite agent
sudo sh -c 'echo "deb https://apt.buildkite.com/buildkite-agent stable main" > /etc/apt/sources.list.d/buildkite-agent.list'
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com --recv-keys 32A37959C2FA5C3C99EFBC32A79206696452D198
sudo apt-get update && sudo apt-get install -y buildkite-agent

# Configure the agent token (from Buildkite dashboard → Agents → Token)
sudo sed -i "s/xxx/YOUR_AGENT_TOKEN/" /etc/buildkite-agent/buildkite-agent.cfg

# Install Azure CLI on the agent
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Start the agent
sudo systemctl enable buildkite-agent
sudo systemctl start buildkite-agent
```

**Option B: Run the agent as a Docker container (quick start)**

```bash
docker run -d \
  --name buildkite-agent \
  --restart unless-stopped \
  -e BUILDKITE_AGENT_TOKEN="YOUR_AGENT_TOKEN" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  buildkite/agent:3
```

**Option C: Use Buildkite Hosted Agents (no infrastructure to manage)**

If you prefer a fully managed option, [Buildkite Hosted Agents](https://buildkite.com/docs/pipelines/hosted-agents) run on Buildkite's infrastructure. You trade the security benefit of self-hosting for operational simplicity.

### Configure secrets on the agent

Create an environment hook on your agent to inject secrets. This hook runs before every build step:

```bash
# /etc/buildkite-agent/hooks/environment
#!/bin/bash
set -euo pipefail

# GHCR credentials for pushing images
export GHCR_USERNAME="your-github-username"
export GHCR_TOKEN="ghp_your_classic_pat_token"

# Azure service principal credentials for deployment
export AZURE_CLIENT_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
export AZURE_CLIENT_SECRET="your-client-secret"
export AZURE_TENANT_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
export AZURE_SUBSCRIPTION_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

Make it executable:
```bash
sudo chmod +x /etc/buildkite-agent/hooks/environment
```

> **Security note:** For production, use a proper secrets manager (Azure Key Vault, AWS Secrets Manager, or HashiCorp Vault) instead of plaintext files. Buildkite provides plugins for each: `aws-sm-buildkite-plugin`, `vault-buildkite-plugin`, etc. The environment hook approach is shown here for simplicity. Buildkite automatically redacts secret values from build logs.

### Write the pipeline

Create `.buildkite/pipeline.yml`:

```yaml
env:
  GHCR_REGISTRY: "ghcr.io"
  IMAGE_PREFIX: "ghcr.io/your-github-username/modern-azure-deploy"

steps:
  # ─── Build and Test ────────────────────────────────
  - group: ":hammer: Build & Test"
    steps:
      - label: ":docker: Build API image"
        key: build-api
        plugins:
          - docker-login#v3.0.0:
              username: "${GHCR_USERNAME}"
              password-env: GHCR_TOKEN
              server: ghcr.io
        command: |
          docker build \
            --target production \
            -t ${IMAGE_PREFIX}-api:${BUILDKITE_BUILD_NUMBER} \
            -t ${IMAGE_PREFIX}-api:${BUILDKITE_COMMIT:0:7} \
            -t ${IMAGE_PREFIX}-api:latest \
            api/
          echo "API image built successfully"
          docker images | grep modern-azure-deploy-api

      - label: ":docker: Build Web image"
        key: build-web
        plugins:
          - docker-login#v3.0.0:
              username: "${GHCR_USERNAME}"
              password-env: GHCR_TOKEN
              server: ghcr.io
        command: |
          docker build \
            --target production \
            -t ${IMAGE_PREFIX}-web:${BUILDKITE_BUILD_NUMBER} \
            -t ${IMAGE_PREFIX}-web:${BUILDKITE_COMMIT:0:7} \
            -t ${IMAGE_PREFIX}-web:latest \
            web/
          echo "Web image built successfully"

      - label: ":test_tube: Run API tests"
        key: test-api
        depends_on: build-api
        command: |
          docker run --rm ${IMAGE_PREFIX}-api:${BUILDKITE_BUILD_NUMBER} \
            python -c "from main import app; print('Import OK')"
          echo "Basic smoke test passed"

  - wait

  # ─── Push to GHCR ─────────────────────────────────
  - group: ":package: Push to GHCR"
    steps:
      - label: ":rocket: Push API image"
        key: push-api
        plugins:
          - docker-login#v3.0.0:
              username: "${GHCR_USERNAME}"
              password-env: GHCR_TOKEN
              server: ghcr.io
        command: |
          docker push ${IMAGE_PREFIX}-api:${BUILDKITE_BUILD_NUMBER}
          docker push ${IMAGE_PREFIX}-api:${BUILDKITE_COMMIT:0:7}
          docker push ${IMAGE_PREFIX}-api:latest
          echo "API images pushed to GHCR"

      - label: ":rocket: Push Web image"
        key: push-web
        plugins:
          - docker-login#v3.0.0:
              username: "${GHCR_USERNAME}"
              password-env: GHCR_TOKEN
              server: ghcr.io
        command: |
          docker push ${IMAGE_PREFIX}-web:${BUILDKITE_BUILD_NUMBER}
          docker push ${IMAGE_PREFIX}-web:${BUILDKITE_COMMIT:0:7}
          docker push ${IMAGE_PREFIX}-web:latest
          echo "Web images pushed to GHCR"

  - wait

  # ─── Deploy to Azure ───────────────────────────────
  - label: ":azure: Deploy to Azure Container Apps"
    key: deploy
    if: build.branch == "main"
    concurrency: 1
    concurrency_group: "azure-production-deploy"
    command: |
      echo "--- :key: Logging in to Azure"
      az login --service-principal \
        -u "$$AZURE_CLIENT_ID" \
        -p "$$AZURE_CLIENT_SECRET" \
        --tenant "$$AZURE_TENANT_ID"
      az account set --subscription "$$AZURE_SUBSCRIPTION_ID"

      IMAGE_TAG="${BUILDKITE_COMMIT:0:7}"

      echo "--- :rocket: Updating API container app"
      az containerapp update \
        --name task-api \
        --resource-group modern-deploy-rg \
        --image ${IMAGE_PREFIX}-api:$${IMAGE_TAG}

      echo "--- :rocket: Updating Web container app"
      az containerapp update \
        --name task-web \
        --resource-group modern-deploy-rg \
        --image ${IMAGE_PREFIX}-web:$${IMAGE_TAG}

      echo "--- :white_check_mark: Deployment complete"
      az containerapp show --name task-web --resource-group modern-deploy-rg \
        --query "properties.configuration.ingress.fqdn" -o tsv
```

### How the pipeline works

The pipeline runs in three phases separated by `wait` steps (synchronization barriers):

1. **Build & Test** — Builds both Docker images in parallel and runs a smoke test. The `docker-login` plugin authenticates to GHCR before each step. Build numbers and git commit SHAs provide unique, traceable tags.

2. **Push to GHCR** — Pushes all tags for both images. This only runs after builds and tests succeed.

3. **Deploy to Azure** — Only triggers on the `main` branch (`if: build.branch == "main"`). Uses **`concurrency: 1`** with a **`concurrency_group`** to prevent two deployments from running simultaneously, which could cause race conditions. Logs into Azure using a service principal and updates both Container Apps.

**The `$$` syntax** in Buildkite pipeline YAML escapes the dollar sign so that environment variables are resolved at runtime on the agent, not during pipeline upload.

### Connect the pipeline to your repository

1. Log in to [buildkite.com](https://buildkite.com)
2. Click **New Pipeline**
3. Connect your GitHub repository
4. Set the initial step command to `buildkite-agent pipeline upload` — this reads `.buildkite/pipeline.yml` from your repo
5. Save and run

Buildkite will trigger builds on every push. The web dashboard shows real-time logs, timing, and status for each step.

> **Common pitfall:** Build steps may execute on different agents. Do not assume Docker images built in one step are available in another. If your agents run on separate machines, push images to GHCR after building and pull them in subsequent steps. On a single agent, built images persist in the local Docker cache across steps within the same build.

---

## Step 8: One-time Azure infrastructure setup

Before Buildkite can deploy, you need to create the Azure resources. Run these commands once from your Codespace, local terminal, or the Buildkite agent.

### Create a service principal for CI/CD

```bash
# Log in to Azure
az login

# Create a service principal with Contributor role on a resource group
az group create --name modern-deploy-rg --location eastus

az ad sp create-for-rbac \
  --name "buildkite-deployer" \
  --role contributor \
  --scopes /subscriptions/$(az account show --query id -o tsv)/resourceGroups/modern-deploy-rg \
  --output json
```

This outputs JSON with `appId` (client ID), `password` (client secret), and `tenant`. Store these in your Buildkite agent's environment hook (Step 7).

### Create the Container Apps environment

```bash
# Register required providers
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights

# Create the shared environment
az containerapp env create \
  --name modern-deploy-env \
  --resource-group modern-deploy-rg \
  --location eastus
```

A Container Apps environment is a secure boundary — apps within it share a virtual network and can communicate via internal DNS. Think of it as a lightweight Kubernetes cluster without the Kubernetes complexity.

### Deploy the API container app

```bash
az containerapp create \
  --name task-api \
  --resource-group modern-deploy-rg \
  --environment modern-deploy-env \
  --image ghcr.io/YOUR_GITHUB_USERNAME/modern-azure-deploy-api:latest \
  --registry-server ghcr.io \
  --registry-username YOUR_GITHUB_USERNAME \
  --registry-password YOUR_GHCR_PAT \
  --target-port 8000 \
  --ingress internal \
  --cpu 0.25 \
  --memory 0.5Gi \
  --min-replicas 1 \
  --max-replicas 3 \
  --env-vars "DB_PATH=/data/tasks.db"
```

**Key flags explained:**

- **`--ingress internal`** makes this service accessible only to other apps in the same environment — not the public internet. The frontend will reach it via its internal FQDN.
- **`--registry-server ghcr.io`** with username and password tells Azure how to authenticate to GHCR to pull private images.
- **`--min-replicas 1`** keeps at least one instance running to avoid cold starts. Set to `0` for scale-to-zero cost savings on non-critical workloads.
- **`--cpu 0.25 --memory 0.5Gi`** is the smallest allocation available, suitable for this lightweight API.

### Deploy the Web container app

```bash
# Get the API's internal FQDN
API_FQDN=$(az containerapp show \
  --name task-api \
  --resource-group modern-deploy-rg \
  --query "properties.configuration.ingress.fqdn" -o tsv)

az containerapp create \
  --name task-web \
  --resource-group modern-deploy-rg \
  --environment modern-deploy-env \
  --image ghcr.io/YOUR_GITHUB_USERNAME/modern-azure-deploy-web:latest \
  --registry-server ghcr.io \
  --registry-username YOUR_GITHUB_USERNAME \
  --registry-password YOUR_GHCR_PAT \
  --target-port 5000 \
  --ingress external \
  --cpu 0.25 \
  --memory 0.5Gi \
  --min-replicas 1 \
  --max-replicas 5 \
  --env-vars "API_URL=https://${API_FQDN}"
```

**`--ingress external`** makes this app publicly accessible. Azure automatically provisions a TLS certificate and gives you a URL like `https://task-web.politecoast-abc123.eastus.azurecontainerapps.io`.

### Verify the deployment

```bash
# Get the public URL
az containerapp show \
  --name task-web \
  --resource-group modern-deploy-rg \
  --query "properties.configuration.ingress.fqdn" -o tsv

# Check both apps are running
az containerapp list \
  --resource-group modern-deploy-rg \
  --output table

# View logs
az containerapp logs show \
  --name task-api \
  --resource-group modern-deploy-rg \
  --follow
```

Expected output:
```
Name       Location  ResourceGroup     Status
---------  --------  ----------------  --------
task-api   eastus    modern-deploy-rg  Running
task-web   eastus    modern-deploy-rg  Running
```

Open the FQDN in your browser. You should see the Task Tracker homepage.

### Azure Container Apps vs Azure Container Instances

The original tutorial likely used Azure Container Instances (ACI). Here is why Container Apps is the better choice for web applications:

| | Container Apps | Container Instances |
|---|---|---|
| **Autoscaling** | Built-in, scale to zero | No autoscaling |
| **TLS/HTTPS** | Automatic managed certificates | Manual configuration |
| **Service discovery** | Built-in between apps | Containers share localhost only |
| **Traffic splitting** | Blue-green and canary deployments | Not available |
| **Cost (idle)** | $0 when scaled to zero | Always billed while running |
| **Free tier** | 180,000 vCPU-seconds/month | None |

ACI is simpler for one-off batch jobs. **Container Apps is the right choice for web services that need autoscaling, HTTPS, and inter-service communication.**

For more on Azure Container Apps, see the [official documentation](https://learn.microsoft.com/en-us/azure/container-apps/).

---

## Step 9: The complete workflow in action

With everything configured, here is what happens when a developer pushes code:

```
Developer pushes to main
        │
        ▼
Buildkite detects the push
        │
        ▼
Agent picks up the job, checks out code
        │
        ├──► Build API image (multi-stage Dockerfile)
        ├──► Build Web image (multi-stage Dockerfile)
        │
        ▼
Run smoke tests on built images
        │
        ▼
Push tagged images to ghcr.io
        │
        ▼
Log in to Azure via service principal
        │
        ▼
Update Container Apps to use new image tags
        │
        ▼
Azure pulls new images from GHCR, performs rolling update
        │
        ▼
New version live at https://task-web.*.azurecontainerapps.io
```

To test this workflow:

```bash
# Make a code change
echo "# Updated $(date)" >> README.md

# Commit and push
git add -A
git commit -m "feat: trigger deployment pipeline"
git push origin main
```

Open the Buildkite dashboard to watch the pipeline execute in real time. Each step shows live logs, duration, and pass/fail status.

---

## Secrets management done right

Secrets flow through several systems in this architecture. Here is where each secret lives and the best practice for each:

- **GHCR PAT (for pushing images)**: Stored in the Buildkite agent's environment hook or a secrets manager. Never committed to the repository. Buildkite automatically redacts recognized secret patterns from logs.
- **Azure service principal credentials**: Stored alongside the GHCR PAT on the agent. For production, use Azure Key Vault integration with the agent.
- **GHCR PAT (for Azure to pull images)**: Passed during `az containerapp create` and stored as a Container Apps registry credential. Azure encrypts this at rest.
- **Application secrets** (database passwords, API keys): Use Azure Container Apps' built-in secrets feature and reference them with `secretref:` syntax in environment variables. For sensitive production workloads, integrate with Azure Key Vault.

```bash
# Example: adding a secret to a Container App
az containerapp secret set \
  --name task-api \
  --resource-group modern-deploy-rg \
  --secrets "my-secret=SuperSecretValue"

# Reference it in environment variables
az containerapp update \
  --name task-api \
  --resource-group modern-deploy-rg \
  --set-env-vars "SECRET_KEY=secretref:my-secret"
```

> **Never put secrets in Docker images, compose files committed to Git, or pipeline YAML.** Use environment-level injection at every layer: agent hooks for CI/CD, Container Apps secrets for runtime, and `.env` files (gitignored) for local development.

---

## Cost considerations and cleanup

Azure Container Apps on the Consumption plan provides **generous free monthly grants** per subscription: 180,000 vCPU-seconds, 360,000 GiB-seconds, and 2 million requests. For a two-app deployment with `0.25 vCPU / 0.5 GiB` each running 24/7, you will stay within or near the free tier during development and light production use. Once you exceed free grants, expect **roughly $5–15/month per container** at this resource level.

Buildkite's free tier supports small teams. GHCR is free for public packages. The biggest cost is the Azure VM running the Buildkite agent (~$15/month for a B2s instance), which you can replace with Buildkite Hosted Agents or shut down when not deploying.

To avoid unexpected charges, clean up when done experimenting:

```bash
# Delete everything in the resource group
az group delete --name modern-deploy-rg --yes --no-wait

# Stop the Buildkite agent VM
az vm deallocate --resource-group my-resource-group --name buildkite-agent
```

---

## Conclusion

This tutorial replaced five manual processes with automated, reproducible alternatives. Multi-stage Dockerfiles cut image sizes and eliminated build tools from production. Docker Compose turned multi-container orchestration into a single declarative file. GHCR placed container images alongside source code with zero additional infrastructure. Buildkite automated the entire build-push-deploy cycle while keeping secrets off third-party servers. And Azure Container Apps provided autoscaling, TLS, and service discovery without Kubernetes complexity.

The most important shift is philosophical: **the GitHub repository becomes the single source of truth for everything** — code, container definitions, environment configuration, and CI/CD pipelines. A new team member opens a Codespace and has a running application in under three minutes. A push to `main` triggers a fully automated deployment. No wiki pages explaining which Docker commands to run. No shared credentials in Slack messages.

Three principles to carry forward as you extend this architecture: tag images with git SHAs (not `latest`) so you can trace every deployment to a commit; use internal ingress for backend services so only the frontend faces the internet; and never store secrets in files that touch version control. These practices scale from a two-container tutorial to a hundred-service production system.
