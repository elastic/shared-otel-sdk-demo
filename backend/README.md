# Backend

A Spring Boot weather service used by EDOT SDK demo applications. It provides a small HTTP API
backed by [Open-Meteo](https://open-meteo.com/) and instruments itself with the
[EDOT Java agent](https://github.com/elastic/elastic-otel-java) via runtime attachment.

## Endpoints

| Method | Path          | Description                              |
|--------|---------------|------------------------------------------|
| GET    | `/v1/health`  | Returns 200 OK when the service is up    |
| GET    | `/v1/forecast?city=<name>` | Returns current temperature for Berlin, London, or Paris. Returns 500 for any other city. |

## Configuration

The following standard OpenTelemetry environment variables are supported at runtime:

| Variable                      | Example                        |
|-------------------------------|--------------------------------|
| `OTEL_SERVICE_NAME`           | `weather-demo-backend`         |
| `OTEL_RESOURCE_ATTRIBUTES`    | `deployment.environment.name=ci` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://localhost:4318`        |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `http/protobuf`                |

## Running with Docker

```shell
docker run --rm -p 8080:8080 \
  -e OTEL_SERVICE_NAME=weather-demo-backend \
  -e OTEL_EXPORTER_OTLP_ENDPOINT=http://host.docker.internal:4318 \
  -e OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf \
  ghcr.io/elastic/shared-otel-sdk-demo/backend:latest
```

## Running with the executable JAR (macOS / no Docker)

```shell
BACKEND_VERSION=1.0.0
BACKEND_JAR="backend-${BACKEND_VERSION}.jar"
BACKEND_RELEASE_URL="https://github.com/elastic/shared-otel-sdk-demo/releases/download/backend/v${BACKEND_VERSION}"

curl -fL -o "${BACKEND_JAR}" "${BACKEND_RELEASE_URL}/${BACKEND_JAR}"
curl -fL -o "${BACKEND_JAR}.sha256" "${BACKEND_RELEASE_URL}/${BACKEND_JAR}.sha256"
shasum -a 256 -c "${BACKEND_JAR}.sha256"

OTEL_SERVICE_NAME=weather-demo-backend \
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318 \
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf \
java -jar "${BACKEND_JAR}"
```

## Building locally

Requires Java 17+.

```shell
./gradlew bootJar
java -jar build/libs/backend.jar
```

## Releasing

Push a tag of the form `backend/vMAJOR.MINOR.PATCH`. The release workflow will:

1. Build the executable JAR and attach it (with a SHA-256 checksum) to a GitHub Release.
2. Build and push the Docker image to `ghcr.io/elastic/shared-otel-sdk-demo/backend`.

```shell
git tag backend/v1.0.0
git push origin backend/v1.0.0
```
