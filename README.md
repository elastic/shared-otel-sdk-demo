# shared-otel-sdk-demo

Shared backend services and tooling used across
[Elastic EDOT SDK](https://github.com/elastic/elastic-otel-java) demo applications.

## Tools

| Tool | Description |
|------|-------------|
| [backend](backend/) | Spring Boot weather service with EDOT Java agent instrumentation |

## Consuming a tool

Each tool is released independently. Releases are tagged `<tool>/vMAJOR.MINOR.PATCH`
(e.g. `backend/v1.0.0`). Each release publishes:

- A Docker image at `ghcr.io/elastic/shared-otel-sdk-demo/<tool>:<version>`
- An executable JAR attached to the corresponding GitHub Release (for environments where Docker is
  unavailable, such as GitHub-hosted macOS runners)

Pin an exact version in consuming repositories so a new release cannot unexpectedly break an older
demo. See each tool's README for usage examples and supported configuration.
