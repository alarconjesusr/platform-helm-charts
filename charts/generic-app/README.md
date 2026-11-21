# generic-app

Reusable Helm chart for deploying an application as a Kubernetes Deployment and
Service. Configure each application through a values file; the chart does not
hardcode application-specific labels, environment variables, secrets, or volumes.

## Install

```sh
helm upgrade --install my-app ./charts/generic-app \
  --namespace dev --create-namespace \
  --values my-app-values.yaml
```

The Helm release namespace is used for the Kubernetes resources.

## Main settings

- `image.repository`, `image.tag`, and `image.pullPolicy` configure the image.
- `replicaCount` sets the number of replicas unless autoscaling is enabled.
- `containerPorts` and `service.ports` accept lists, so an application can expose
  multiple ports.
- `commonLabels` are added to the Deployment and Pods; `podLabels` and
  `podAnnotations` apply only to Pods.
- `env`, `volumeMounts`, `volumes`, `resources`, and `imagePullSecrets` map to
  their corresponding Kubernetes container or Pod fields.
- `configMap.enabled`, `autoscaling.enabled`, `pdb.enabled`, `ingress.enabled`,
  and `httpRoute.enabled` control optional resources. They are disabled by
  default.

Environment values can use Kubernetes `valueFrom` references. Store sensitive
values in Kubernetes Secrets instead of placing their contents in this chart's
values files.

Example:

```yaml
replicaCount: 3
image:
  repository: registry.example.net/team/my-app
  tag: "1.0.0"
  pullPolicy: Always

commonLabels:
  tags.datadoghq.com/env: dev
  tags.datadoghq.com/service: my-app

containerPorts:
  - name: http
    containerPort: 5000
  - name: metrics
    containerPort: 5001

service:
  ports:
    - name: http
      port: 8080
      targetPort: http

env:
  - name: APP_CONNECTION
    valueFrom:
      secretKeyRef:
        name: my-app-secrets
        key: connection

resources:
  requests:
    cpu: 100m
    memory: 64Mi
  limits:
    cpu: "1"
    memory: 2Gi
```

Ingress and HTTPRoute require a matching controller/API in the cluster. HTTPRoute
also requires Gateway API resources and suitable `parentRefs`. The chart's
`helm lint` check can be run with `helm lint ./charts/generic-app`.
