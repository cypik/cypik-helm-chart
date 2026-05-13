# Cypik Helm Chart

Reusable Helm chart for deploying most client/project workloads: web apps, APIs, background workers, and scheduled jobs.

## Install

```bash
helm upgrade --install my-app ./charts/cypik -n my-namespace --create-namespace
```

## Common Client Override

Create a separate values file per client/project:

```bash
cp charts/cypik/values.yaml values-client-a.yaml
helm upgrade --install client-a-api ./charts/cypik -n client-a -f values-client-a.yaml
```

## What It Supports

- Deployment with configurable image, command, args, env, ports, resources, probes, sidecars, and init containers
- Service with configurable ports and type
- Ingress with class, annotations, hosts, paths, and TLS
- ConfigMap and Secret injection through `envFrom`
- ServiceAccount, pod security context, node selectors, tolerations, affinity
- Optional RBAC Role/ClusterRole and NetworkPolicy
- HorizontalPodAutoscaler
- KEDA ScaledObject
- PodDisruptionBudget
- PersistentVolumeClaim
- CronJobs
- Multiple apps/sites in one release with `applications`
- Extra raw Kubernetes objects through `extraObjects`
- Documented `values.yaml` and `values.schema.json` for safer public chart usage

## Minimal Values

```yaml
image:
  repository: ghcr.io/company/my-app
  tag: "1.0.0"

containerPorts:
  - name: http
    containerPort: 3000

service:
  ports:
    - name: http
      port: 80
      targetPort: http
```

## Enable Only What You Need

Most resources are optional. Keep their `enabled` value `false` unless that client/project needs them.

```yaml
deployment:
  enabled: true

service:
  enabled: true

ingress:
  enabled: false

autoscaling:
  enabled: false

keda:
  enabled: false

persistence:
  enabled: false

config:
  enabled: false

secret:
  enabled: false

rbac:
  create: false

networkPolicy:
  enabled: false

pdb:
  enabled: false

tests:
  enabled: false
```

For a worker without a Service:

```yaml
service:
  enabled: false
```

## Ingress Example

```yaml
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
          servicePort: http
  tls:
    - secretName: app-example-com-tls
      hosts:
        - app.example.com
```

## KEDA ScaledObject Example

Use this when KEDA is installed in the cluster and you want trigger-based scaling. Keep normal `autoscaling.enabled` false because KEDA creates its own HPA.

```yaml
keda:
  enabled: true
  pollingInterval: 30
  cooldownPeriod: 300
  minReplicaCount: 0
  maxReplicaCount: 10
  triggers:
    - type: cpu
      metricType: Utilization
      metadata:
        value: "70"
```

## Multiple Apps Example

Use `applications` when one Helm release should deploy multiple sites, APIs, or workers.

```yaml
deployment:
  enabled: false
service:
  enabled: false

applications:
  - name: frontend
    image:
      repository: ghcr.io/company/frontend
      tag: "1.0.0"
    containerPorts:
      - name: http
        containerPort: 3000
    service:
      enabled: true
      ports:
        - name: http
          port: 80
          targetPort: http
    ingress:
      enabled: true
      className: nginx
      hosts:
        - host: app.example.com
          paths:
            - path: /
              pathType: Prefix
              servicePort: http

  - name: api
    image:
      repository: ghcr.io/company/api
      tag: "1.0.0"
    containerPorts:
      - name: http
        containerPort: 8080
    service:
      enabled: true
      ports:
        - name: http
          port: 80
          targetPort: http
```

## Validate

```bash
helm lint ./charts/cypik
helm template my-app ./charts/cypik
```

## Publish Notes

For Helm repositories and Artifact Hub, keep `values.yaml`, `Chart.yaml`, `README.md`, and `values.schema.json` at the chart root. In this repository, that chart root is `charts/cypik`. The `templates/` directory should contain only rendered Kubernetes manifests and helper templates.
