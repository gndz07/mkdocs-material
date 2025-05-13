---
template: overrides/main.html
---

# Embedding content from a private repository

You can embed content from a private repository by setting the `GH_TOKEN` environment variable to a personal access token with read access to the repository.

The following example is an embedded content from a private repository.

## Quick start guide on Kubernetes
{{ external_markdown('https://raw.githubusercontent.com/traefik/hub-doc/refs/heads/main/docs/api-gateway/quick-start-guide.md', '## On Kubernetes') }}