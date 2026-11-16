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

## Conjur with Secrets Store CSI Driver

Set `conjur.enabled` to let the chart create a `SecretProviderClass`, add the
Conjur secret annotation, and mount the CSI volume. The CSI driver and
CyberArk's [Conjur CSI provider](https://github.com/cyberark/conjur-k8s-csi-provider)
must already be installed in the cluster. Conjur must be configured to
authenticate the workload's Kubernetes ServiceAccount (for example, with
Authn JWT). The application reads each secret from a file; the chart does not
create a Kubernetes Secret or put secret values in Helm values.

```yaml
serviceAccount:
  create: true
  name: my-app

conjur:
  enabled: true
  account: myAccount
  applianceUrl: https://conjur.example.net
  authnId: authn-jwt/kubernetes
  # Required by the provider; supply the Conjur appliance CA certificate.
  sslCertificate: |
    -----BEGIN CERTIFICATE-----
    <Conjur appliance CA certificate>
    -----END CERTIFICATE-----
  secretProviderClassName: app-conjur-secrets
  mountPath: /mnt/secrets-store
  secrets:
    - file: db-password
      variable: db-credentials/password
```

The application reads the variable from
`/mnt/secrets-store/db-password`. `conjur.secrets` maps relative file paths to
Conjur variable IDs. The chart generates the `SecretProviderClass` in the
release namespace and sets `conjur.org/secrets` on the Pod.

The Conjur policy must authorize the identity
`system:serviceaccount:<namespace>:my-app` to authenticate and read the
requested variable. The CSI driver must request the JWT audience configured
for the Conjur authenticator (the provider's installation guide uses
`conjur`). See the provider's [installation and configuration
guide](https://github.com/cyberark/conjur-k8s-csi-provider#readme) for
authenticator policy and version-specific details.

For a complete local end-to-end procedure using the included Conjur OSS
Compose environment and Minikube, see
[`examples/conjur-local/README.md`](../../../examples/conjur-local/README.md).
Conjur stays in Compose; only the CSI components and the application Pod run in
Kubernetes. `helm lint` and `helm template` only validate chart rendering; the
end-to-end test must verify that a running Pod can read the mounted file.

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
