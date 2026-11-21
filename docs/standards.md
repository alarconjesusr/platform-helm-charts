# Chart standards

## Structure

- Keep each reusable workload type in its own directory under `charts/`.
- Put chart-specific defaults in that chart's `values.yaml`.
- Keep Kubernetes resources in named templates under `templates/`.
- Document chart options and examples in the chart's `README.md`.

## Reusability and configuration

- Do not hardcode application names, environment names, registry credentials,
  secret contents, or environment-specific URLs in templates.
- Use Helm release/chart helpers for resource names and stable selector labels.
- Keep selectors consistent between Deployments, Services, and PodDisruptionBudgets.
- Expose application-specific labels, annotations, ports, environment variables,
  volumes, and resources through values.
- Optional resources must have an explicit `enabled` value and be disabled by
  default unless the chart documents otherwise.

## Workload practices

- Prefer explicit image tags and `imagePullPolicy` over `latest`.
- Configure resource requests and limits for production workloads.
- Reference sensitive data from Kubernetes Secrets; do not commit credentials
  or put secret values in ConfigMaps.
- Keep chart defaults safe to render in a development cluster. Environment- or
  cluster-specific settings belong in a separate values file.

## Validation

Run `helm lint ./charts/<chart-name>` and render representative values with
`helm template` before submitting changes.
