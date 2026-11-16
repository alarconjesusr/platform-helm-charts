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

The release workflow publishes versioned charts to GitHub Releases and serves
the Helm repository index from GitHub Pages:
`https://alarconjesusr.github.io/platform-helm-charts`.

```sh
helm repo add platform-helm-charts https://alarconjesusr.github.io/platform-helm-charts
helm repo update

# Download the packaged chart to the current directory.
helm pull platform-helm-charts/generic-app --version 0.1.1

# Or install it directly into a Kubernetes cluster.
helm upgrade --install my-app platform-helm-charts/generic-app \
  --version 0.1.1 \
  --namespace dev --create-namespace \
  --values my-app-values.yaml
```

For chart design rules, versioning, and contribution steps, see [`docs/`](./docs/).
