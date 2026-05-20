# Cypik Helm Chart

A reusable Helm chart for deploying single or multiple Kubernetes applications from one release. It supports common application resources such as Deployments, Services, Ingress, ConfigMaps, Secrets, PVCs, StorageClasses, RBAC, HPA, KEDA ScaledObjects, PDBs, NetworkPolicies, Jobs, CronJobs, and custom extra objects.

## Features

- Single-application and multi-application deployment modes
- Per-application Deployment, Service, Ingress, ConfigMap, Secret, PVC, PDB, NetworkPolicy, HPA, KEDA, RBAC, and Job support
- Optional cluster-scoped StorageClass creation
- Optional Helm test pod
- Support for init containers, sidecars, probes, resources, security contexts, node selectors, tolerations, affinity, volumes, and extra objects
- JSON schema support for values validation

## Chart Structure

```text
.
├── Chart.yaml
├── values.yaml
├── values.schema.json
├── templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── pvc.yaml
│   ├── storageclass.yaml
│   ├── networkpolicy.yaml
│   ├── rbac.yaml
│   ├── hpa.yaml
│   ├── scaledobject.yaml
│   ├── pdb.yaml
│   ├── jobs.yaml
│   ├── cronjobs.yaml
│   ├── extra-objects.yaml
│   └── tests
└── README.md
```

## Validate the Chart

Render the manifests locally:

```bash
helm template cypik .
```

Run Helm lint:

```bash
helm lint .
```

If you want to test without creating a custom StorageClass:

```bash
helm template cypik . --set storageClass.enabled=false
```

## Install

Install into the current namespace:

```bash
helm upgrade --install cypik .
```

Install into a dedicated namespace:

```bash
helm upgrade --install cypik . \
  --namespace cypik \
  --create-namespace
```

## Multi-Application Example

Use the `applications` array to deploy multiple applications in the same Helm release.

```yaml
applications:
  - name: frontend
    image:
      repository: nginx
      tag: "1.25"
    service:
      enabled: true
      ports:
        - name: http
          port: 80
          targetPort: 80
    ingress:
      enabled: true
      className: alb
      annotations:
        alb.ingress.kubernetes.io/scheme: internet-facing
        alb.ingress.kubernetes.io/target-type: ip
      hosts:
        - host: app.example.com
          paths:
            - path: /
              pathType: Prefix
              servicePort: http

  - name: api
    image:
      repository: my-api
      tag: "1.0.0"
    service:
      enabled: true
      ports:
        - name: http
          port: 80
          targetPort: 8080
    env:
      - name: APP_ENV
        value: production
```

## ServiceAccount and RBAC

Create a dedicated ServiceAccount for an application:

```yaml
applications:
  - name: api
    serviceAccount:
      create: true
    rbac:
      create: true
      clusterWide: false
      rules:
        - apiGroups: [""]
          resources: ["pods"]
          verbs: ["get", "list"]
```

Use an existing ServiceAccount:

```yaml
applications:
  - name: api
    serviceAccount:
      create: false
      name: default
```

When enabling RBAC, set `serviceAccount.create: true` or provide `serviceAccount.name` explicitly so the RoleBinding or ClusterRoleBinding grants permissions to the same ServiceAccount used by the Pod.

## Configuration, Secrets, and Environment Variables

Create a ConfigMap and inject it with `envFrom`:

```yaml
applications:
  - name: api
    config:
      enabled: true
      data:
        APP_NAME: api
        LOG_LEVEL: info
```

Create a Secret and inject it with `envFrom`:

```yaml
applications:
  - name: api
    secret:
      enabled: true
      stringData:
        DATABASE_URL: postgres://user:password@postgres:5432/app
```

Use direct environment variables:

```yaml
applications:
  - name: api
    env:
      - name: APP_ENV
        value: production
```

Use Kubernetes field references with `envRaw`:

```yaml
applications:
  - name: api
    envRaw:
      - name: POD_IP
        valueFrom:
          fieldRef:
            fieldPath: status.podIP
```

Use an existing ConfigMap or Secret:

```yaml
applications:
  - name: api
    envFrom:
      - configMapRef:
          name: existing-config
      - secretRef:
          name: existing-secret
```

Avoid storing production secrets directly in values files. Prefer External Secrets, sealed secrets, or a managed secret store.

## Probes, Resources, and Lifecycle

Enable health checks:

```yaml
applications:
  - name: api
    probes:
      readiness:
        enabled: true
        httpGet:
          path: /health
          port: http
        initialDelaySeconds: 5
        periodSeconds: 10
      liveness:
        enabled: true
        httpGet:
          path: /health
          port: http
        initialDelaySeconds: 15
        periodSeconds: 20
```

Set CPU and memory requests and limits:

```yaml
applications:
  - name: api
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi
```

Configure lifecycle hooks:

```yaml
applications:
  - name: api
    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "sleep 10"]
```

## Security Contexts

Set Pod and container security contexts:

```yaml
applications:
  - name: api
    podSecurityContext:
      runAsNonRoot: true
      fsGroup: 1000
    securityContext:
      allowPrivilegeEscalation: false
      runAsUser: 1000
      runAsGroup: 1000
      capabilities:
        drop:
          - ALL
```

Use stricter settings only after confirming the image supports non-root execution.

## Scheduling Controls

Use node selectors:

```yaml
applications:
  - name: api
    nodeSelector:
      kubernetes.io/os: linux
```

Use tolerations:

```yaml
applications:
  - name: worker
    tolerations:
      - key: dedicated
        operator: Equal
        value: workers
        effect: NoSchedule
```

Use affinity:

```yaml
applications:
  - name: api
    affinity:
      podAntiAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app.kubernetes.io/component: api
              topologyKey: kubernetes.io/hostname
```

Use topology spread constraints:

```yaml
applications:
  - name: api
    topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app.kubernetes.io/component: api
```

## Persistent Volumes

Enable a PVC for an application:

```yaml
applications:
  - name: frontend
    persistence:
      enabled: true
      size: 1Gi
      mountPath: /usr/share/nginx/html
```

When `persistence.storageClass` is empty, Kubernetes uses the cluster default StorageClass.

Use a specific existing StorageClass:

```yaml
applications:
  - name: frontend
    persistence:
      enabled: true
      size: 10Gi
      storageClass: gp3
      mountPath: /data
```

## Optional StorageClass

By default, prefer the cluster default StorageClass and keep custom StorageClass creation disabled:

```yaml
storageClass:
  enabled: false
```

Create a StorageClass only when required:

```yaml
storageClass:
  enabled: true
  name: cypik-gp3
  provisioner: ebs.csi.aws.com
  reclaimPolicy: Retain
  volumeBindingMode: WaitForFirstConsumer
  allowVolumeExpansion: true
  parameters:
    type: gp3
```

StorageClass is cluster-scoped. Use unique names and avoid enabling it by default across multiple releases.

## Extra Volumes, Init Containers, and Sidecars

Mount an additional existing Secret:

```yaml
applications:
  - name: api
    volumes:
      - name: app-cert
        secret:
          secretName: app-cert
    volumeMounts:
      - name: app-cert
        mountPath: /etc/certs
        readOnly: true
```

Run an init container before the main container:

```yaml
applications:
  - name: api
    initContainers:
      - name: wait-for-db
        image: busybox:1.36
        command: ["sh", "-c", "sleep 10"]
```

Add a sidecar:

```yaml
applications:
  - name: api
    sidecars:
      - name: metrics-sidecar
        image: busybox:1.36
        command: ["sh", "-c", "while true; do sleep 30; done"]
```

## NetworkPolicy

Deny all ingress traffic to an application:

```yaml
applications:
  - name: frontend
    networkPolicy:
      enabled: true
      policyTypes:
        - Ingress
      ingress: []
```

Allow only pods with `app=allowed-client` to reach port `80`:

```yaml
applications:
  - name: frontend
    networkPolicy:
      enabled: true
      policyTypes:
        - Ingress
      ingress:
        - from:
            - podSelector:
                matchLabels:
                  app: allowed-client
          ports:
            - protocol: TCP
              port: 80
```

Test from a blocked client:

```bash
kubectl run np-client --image=busybox:1.36 -- sleep 3600
kubectl exec np-client -- wget -qO- --timeout=5 http://<release>-cypik-frontend
```

Expected result for a denied client is a timeout.

Test from an allowed client:

```bash
kubectl run allowed-client --image=busybox:1.36 --labels=app=allowed-client -- sleep 3600
kubectl exec allowed-client -- wget -qO- --timeout=5 http://<release>-cypik-frontend
```

Expected result is an application response. An HTTP error such as `403 Forbidden` still means NetworkPolicy allowed the traffic and the application responded.

On EKS, ensure network policy enforcement is enabled in the CNI or through a compatible policy engine such as Calico.

## PodDisruptionBudget

Use a PDB to keep at least one pod available during voluntary disruptions:

```yaml
applications:
  - name: api
    pdb:
      enabled: true
      minAvailable: 1
```

Or limit unavailable pods:

```yaml
applications:
  - name: api
    pdb:
      enabled: true
      maxUnavailable: 1
```

## Autoscaling

Enable Kubernetes HPA:

```yaml
applications:
  - name: api
    autoscaling:
      enabled: true
      minReplicas: 2
      maxReplicas: 10
      targetCPUUtilizationPercentage: 70
```

Enable KEDA:

```yaml
applications:
  - name: worker
    keda:
      enabled: true
      minReplicaCount: 1
      maxReplicaCount: 10
      triggers:
        - type: cpu
          metricType: Utilization
          metadata:
            value: "70"
```

Do not enable HPA and KEDA for the same application at the same time.

## Jobs and CronJobs

Run a one-off Helm hook Job:

```yaml
applications:
  - name: api
    jobs:
      - name: db-migrate
        enabled: true
        command: ["python", "manage.py", "migrate"]
```

Run a CronJob:

```yaml
cronjobs:
  - name: cleanup
    schedule: "0 2 * * *"
    image:
      repository: busybox
      tag: "1.36"
    command: ["sh", "-c", "echo cleanup"]
```

## Extra Objects

Use `extraObjects` for Kubernetes resources that are not covered by built-in templates.

```yaml
extraObjects:
  - apiVersion: v1
    kind: ConfigMap
    metadata:
      name: extra-config
    data:
      key: value
```

Values in extra objects are rendered with Helm `tpl`, so chart values can be referenced when needed.

## Helm Tests

Enable the built-in Helm test pod for single-app mode:

```yaml
tests:
  enabled: true
```

Run tests:

```bash
helm test <release>
```

## Upgrade and Uninstall

Upgrade an existing release:

```bash
helm upgrade cypik . -f values.yaml
```

Uninstall a release:

```bash
helm uninstall cypik
```

PVC deletion depends on the StorageClass reclaim policy and Helm ownership. Confirm data retention behavior before uninstalling production releases.

## Troubleshooting

Check rendered manifests:

```bash
helm template cypik . --debug
```

Check release resources:

```bash
kubectl get all
kubectl get ingress
kubectl get pvc
kubectl get networkpolicy
```

Check rollout status:

```bash
kubectl rollout status deployment/<release>-cypik-frontend
```

Check a pod's ServiceAccount:

```bash
kubectl get pod <pod-name> -o jsonpath='{.spec.serviceAccountName}'
```

Check PVC binding:

```bash
kubectl get pvc
kubectl describe pvc <pvc-name>
```

Check NetworkPolicy behavior:

```bash
kubectl describe networkpolicy <policy-name>
kubectl run np-client --image=busybox:1.36 -- sleep 3600
kubectl exec np-client -- wget -qO- --timeout=5 http://<service-name>
```

## Production Checklist

- Pin image tags. Avoid `latest`.
- Set CPU and memory requests and limits.
- Enable readiness and liveness probes.
- Use explicit ServiceAccount settings when RBAC is enabled.
- Use external secret management for production secrets.
- Keep `storageClass.enabled=false` unless the release should manage a cluster-scoped StorageClass.
- Confirm NetworkPolicy enforcement is active in the cluster before relying on policies.
- Test PVC binding and persistence in a staging namespace.
- Validate Ingress annotations and hostnames for the target ingress controller.

## License

MIT License. See [LICENSE](LICENSE) for details.
