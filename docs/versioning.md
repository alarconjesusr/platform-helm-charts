# Versioning

Charts follow Semantic Versioning. The chart version is the `version` field in
`Chart.yaml`; increment it whenever chart templates, defaults, or chart
documentation change:

- **Patch**: backwards-compatible fixes or default corrections.
- **Minor**: backwards-compatible options or resources.
- **Major**: breaking changes to values, resource names, or chart behavior.

`appVersion` describes the application version and is independent of the chart
version. In `generic-app`, it is also the fallback image tag when
`image.tag` is empty. Set an explicit `image.tag` in application values when
that fallback is not the desired image version.

The release workflow watches chart changes on the repository's default branch.
When a chart's version has not been released before, it packages the chart and
publishes a GitHub Release and Helm index to the `gh-pages` branch. Bump
`Chart.yaml`'s `version` to publish an update; do not reuse an already released
chart version.

Consumers should pin chart versions in deployment automation and use immutable
application image tags for reproducible deployments.
