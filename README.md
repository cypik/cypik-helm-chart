# Cypik Helm Charts

This repository contains reusable Helm charts.

## Structure

```text
.
├── charts
│   └── cypik
│       ├── Chart.yaml
│       ├── charts
│       ├── templates
│       ├── values.schema.json
│       └── values.yaml
├── LICENSE
└── README.md
```

## Install

```bash
helm template . -f values.yaml
```

## 📁 Chart Templates Overview

This chart consists of several templates, each serving a specific purpose in your Kubernetes cluster:

| Template | Purpose | When to use |
| :--- | :--- | :--- |
| `deployment.yaml` | Manages Pods and ReplicaSets. | Use for stateless applications like APIs and Frontends. |
| `service.yaml` | Exposes your app internally or externally. | Use when you need to enable networking between pods or from outside. |
| `ingress.yaml` | Manages external access (HTTP/HTTPS) to services. | Use for web-accessible applications with custom domains. |
| `configmap.yaml` | Stores non-confidential configuration data. | Use for environment variables and app settings. |
| `secret.yaml` | Stores sensitive data like passwords/keys. | Use for database credentials or API tokens. |
| `jobs.yaml` | Runs one-off tasks (Hooks). | Use for Database Migrations or initial setup tasks. |
| `cronjobs.yaml` | Runs recurring tasks on a schedule. | Use for daily backups, cleanup tasks, or report generation. |
| `hpa.yaml` / `scaledobject.yaml` | Manages Horizontal Pod Autoscaling. | Use when you want your app to scale based on CPU/Memory or KEDA triggers. |
| `pdb.yaml` | Ensures high availability during maintenance. | Use to prevent too many pods from being down at once. |
| `pvc.yaml` | Manages Persistent Volume Claims. | Use when your application needs persistent storage (Stateful apps). |
| `rbac.yaml` | Manages Roles and RoleBindings. | Use when your pods need specific permissions within the cluster. |
| `extra-objects.yaml` | Supports any custom K8s resource. | Use for resources not explicitly covered by standard templates. |

## 🛠 Configuration Examples

### 1. Multi-Application Mode
Deploy multiple services independently using the `applications` array:

```yaml
applications:
  - name: frontend
    image:
      repository: my-frontend
    service:
      enabled: true
  - name: api
    image:
      repository: my-api
    config:
      enabled: true
      data:
        APP_NAME: "api-service"
```

### 2. One-off Jobs (DB Migrations)
Run jobs during installation or upgrade using Helm hooks:

```yaml
applications:
  - name: api
    jobs:
      - name: db-migrate
        enabled: true
        command: ["python", "manage.py", "migrate"]
```

## 📖 Parameters

A complete list of parameters can be found in `values.yaml`. For the best experience, use an editor like VS Code to edit `values.yaml`; you will receive real-time suggestions and documentation for every field.

## 📄 License
MIT License. See [LICENSE](LICENSE) for details.
