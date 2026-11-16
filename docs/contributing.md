# Contributing

1. Make changes in the relevant chart directory and keep application-specific
   values out of reusable templates.
2. Update the chart README or this documentation when behavior or configuration
   changes.
3. Increment the chart `version` for changes intended to be released, following
   [versioning.md](./versioning.md).
4. Validate the chart:

   ```sh
   helm lint ./charts/<chart-name>
   helm template test ./charts/<chart-name>
   ```

5. Open a pull request with a summary of the behavior change and validation
   results.

The Helm lint workflow runs on pushes and pull requests. After a versioned chart
change is merged to the default branch, the release workflow publishes it to
GitHub Pages.
Do not include credentials or secret values in commits, examples, or generated
test output.
