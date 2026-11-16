# Helm charts

Reusable Helm charts for Kubernetes workloads.

## Charts

- [`generic-app`](./charts/generic-app/README.md): configurable application
  Deployment and Service, with optional ConfigMap, HPA, PDB, Ingress, and
  Gateway API HTTPRoute resources.

## Install from source

```sh
helm upgrade --install my-app ./charts/generic-app \
  --namespace dev --create-namespace \
  --values my-app-values.yaml
```

See the chart's README and `values.yaml` for configuration options.

## Validate

```sh
helm lint ./charts/generic-app
helm template my-app ./charts/generic-app --values my-app-values.yaml
```

## Published Helm repository

The release workflow publishes versioned charts and creates the `gh-pages`
branch. After the first successful release, enable GitHub Pages for that branch
in the repository settings, then add the repository:

```sh
helm repo add generic-charts https://<owner>.github.io/<repository>
helm repo update
helm install my-app generic-charts/generic-app --values my-app-values.yaml
```

For chart design rules, versioning, and contribution steps, see [`docs/`](./docs/).
