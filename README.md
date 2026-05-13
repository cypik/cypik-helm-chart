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
helm upgrade --install my-app ./charts/cypik -n my-namespace --create-namespace
```

## Validate

```bash
helm lint ./charts/cypik
helm template my-app ./charts/cypik
```

## Package

```bash
helm package ./charts/cypik --destination /tmp
```

The chart documentation and values are inside [charts/cypik/README.md](charts/cypik/README.md).
