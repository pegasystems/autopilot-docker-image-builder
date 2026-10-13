# Custom Autopilot Service Images

> [!IMPORTANT]
> **Match the Python base image to the Autopilot source-image version.** The
> Python base image is not fixed for all Autopilot releases and must be selected
> according to the source image tag:
>
> | Autopilot version | Required Python version |
> | --- | --- |
> | `<= 2.42` | Python `3.13` |
> | `> 2.42` | Python `3.14` |
>
> Do not use a floating Python tag such as `python:3.13-alpine` or
> `python:3.14-alpine`. Resolve and pin a compatible patch version or digest.
> Always verify the selected Python version against the source image's
> `constraints.txt` before building. A mismatch can cause packages to be
> skipped by environment markers or fail during installation. Follow this
> README's compatibility guidance whenever the Autopilot source-image tag
> changes.

Build client-specific Autopilot Service images using the same two-stage pattern
as the SRS Docker image builder:

1. Pull a released Pega image as an artifact source.
2. Build the final image from a client-approved runtime base image.
3. Copy only the runtime artifacts required by the service.

This repository supports both images required by the Autopilot deployment:

- `Dockerfile.autopilot` builds the primary Python service image.
- `Dockerfile.mcp` builds the Infinity Rules MCP Java image.

The client can use any approved base-image registry and distribution. The default
values are local examples only and must be replaced for production use.

The source image is also configurable. For local testing, it can point to an
already downloaded/tagged image instead of Pega Downloads.

## Prerequisites

- Docker with Buildx support for multi-platform images.
- Credentials for `pega-docker.downloads.pega.com`.
- A released Autopilot image version.
- An approved final runtime base for each image.
- Credentials for the destination private container registry.

Authenticate to Pega Downloads without placing a password in shell history:

```bash
printf '%s' "$PEGA_DOWNLOADS_PASSWORD" | \
  docker login pega-docker.downloads.pega.com \
  --username "$PEGA_DOWNLOADS_USERNAME" --password-stdin
```

Authenticate to the client base-image registry when it is private:

```bash
docker login <client-base-registry>
```

## Required Base Images

The client must provide approved immutable references, preferably including a
digest:

```text
Autopilot base: <registry>/<repository>:<tag>@sha256:<digest>
MCP base:       <registry>/<repository>:<tag>@sha256:<digest>
```

The Autopilot base must provide:

- A Python version supported by the released constraints. The tested `2.42.0`
  source image uses Python `3.13.13` and its constraints reject Python versions
  newer than `3.13.15`. Autopilot versions above `2.42` use Python `3.14`.
- `pip` for installing the locked dependencies.
- `apt-get` and a POSIX-compatible shell if using the supplied Dockerfile unchanged.
- CA certificates.

The MCP base must provide:

- Java/JRE 17.
- `apk` and a POSIX-compatible shell if using the supplied Dockerfile unchanged.
- CA certificates.

If the client uses a different package manager or shell, adapt the small setup
sections in the Dockerfiles. Do not silently substitute an unapproved base.

## Build Autopilot

The released image supplies these artifacts:

```text
/lean/constraints.txt
/lean/main
/lean/autopilotstorage
/lean/autopilots_v2
/lean/entrypoint.sh
```

Dependencies are installed again on the client base. Python site-packages are not
copied from the Pega image because native modules may depend on its libc, Python
patch version, or shared libraries.

Single-platform build:

```bash
docker build \
  --file Dockerfile.autopilot \
  --build-arg AUTOPILOT_SOURCE_IMAGE='docker-dev.bin.pega.io/platformservices/autopilot-service:2.42.0-branch-33405873755-20260831150027' \
  --build-arg AUTOPILOT_BASE_IMAGE='python:3.13.13-slim' \
  --tag <private-registry>/platformservices/autopilot-service:2.42.0-custom.1 \
  .
```

Multi-platform build and publish:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --file Dockerfile.autopilot \
  --build-arg AUTOPILOT_SOURCE_IMAGE='<released-autopilot-image>' \
  --build-arg AUTOPILOT_BASE_IMAGE='<client-python-image>@sha256:<digest>' \
  --tag <private-registry>/platformservices/autopilot-service:2.42.0-custom.1 \
  --push \
  .
```

## Build MCP

The released MCP image supplies:

```text
/opt/mcp/infinity-rules-mcp.jar
/opt/infinity-skills/
```

Build the companion image from the client-approved Java base:

```bash
docker build \
  --file Dockerfile.mcp \
  --build-arg MCP_SOURCE_IMAGE='<released-mcp-image>' \
  --build-arg MCP_BASE_IMAGE='<client-jre-image>@sha256:<digest>' \
  --tag <private-registry>/platformservices/infinity-rules-skills-mcp:2.44.0-custom.1 \
  .
```

Both images should be built from compatible Autopilot release versions.

### Local 2.42.0 MCP Build

The MCP source image can be supplied from a locally downloaded image. This
build was tested using the Java 17 Alpine runtime base:

```bash
docker build \
  --file Dockerfile.mcp \
  --build-arg MCP_SOURCE_IMAGE='docker-dev.bin.pega.io/platformservices/infinity-rules-skills-mcp:2.42.0-20260908160327' \
  --build-arg MCP_BASE_IMAGE='eclipse-temurin:17-jre-alpine' \
  --tag infinity-rules-skills-mcp:2.42.0-20260908160327-jre17-alpine-custom \
  .
```

The resulting custom image was 306 MB, compared with the 440 MB source image.
Run it with Docker options before the image reference so Docker publishes the
MCP port to the host:

```bash
docker run --detach --rm \
  --name mcp-2420-custom \
  --publish 8090:8090 \
  infinity-rules-skills-mcp:2.42.0-20260908160327-jre17-alpine-custom
```

Validate the stateless MCP endpoint from the host. Use `127.0.0.1`, not
`0.0.0.0`, as the client target:

```bash
curl --fail --show-error --silent \
  --request POST http://127.0.0.1:8090/mcp \
  --header 'Content-Type: application/json' \
  --header 'Accept: text/event-stream, application/json' \
  --data '{"jsonrpc":"2.0","id":0,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"healthcheck","version":"1.0"}}}'
```

## Python 3.13 Test Result

The following build was tested locally using the existing downloaded source image:

```text
Source: docker-dev.bin.pega.io/platformservices/autopilot-service:2.42.0-branch-33405873755-20260831150027
Base:   python:3.13.13-slim
Result: image build succeeded; container started; GET /v1/health returned true
```

The floating `python:3.13-alpine` tag currently resolves to Python `3.13.16`,
which is outside this release's constraints. Pinning `python:3.13.13-alpine`
corrected the version-marker issue, but the build still failed at
`polars==0.20.3`: the package has no compatible Alpine wheel for this build and
its Rust source failed with the current Alpine toolchain. Therefore Alpine is
not certified for this Autopilot release by this test. An Alpine client base
requires a separately validated package/toolchain strategy.

The supplied Dockerfile installs native build tools only during dependency
installation and removes them before the final image is assembled. This is
required because some locked packages do not provide a compatible prebuilt wheel
for every platform/base combination.

## Deploying the Images

The Autopilot Helm/SCE deployment uses these image paths:

```text
<private-registry>/platformservices/autopilot-service:2.44.0-custom.1
<private-registry>/platformservices/infinity-rules-skills-mcp:2.44.0-custom.1
```

Configure the main service values as:

```text
ServiceDockerImageRepo=<private-registry>
ServiceDockerImageTag=2.44.0-custom.1
```

The MCP image uses the corresponding repository and tag through the deployment’s
MCP image configuration. Keep the repository suffixes unless the Helm/SCE
configuration is changed accordingly.

## Supported Customizations

Add organization-specific configuration, certificates, or other approved files
after the base runtime is established. For example:

```dockerfile
USER root
COPY --chown=leanusr:leanusr models.json /config/models.json
USER leanusr
ENV LOCAL_MODELS_FILE=/config/models.json
```

Do not add secrets, API keys, tokens, or private keys to the image or build
context. Pass them at runtime through Kubernetes Secrets or a secrets manager.

## Runtime Contract

Do not change these paths or commands unless the service has been revalidated:

- `/lean/main`
- `/lean/autopilotstorage`
- `/lean/autopilots_v2`
- `/lean/entrypoint.sh`
- Autopilot port `8080`
- MCP port `8090`
- Autopilot non-root user `leanusr`
- MCP non-root user with UID `1001`

The Launchpad skills directory is not copied separately by these Dockerfiles. The
primary Autopilot application image does not require that overlay for the normal
service runtime. If a deployment explicitly enables a feature that requires
Launchpad skills, the image must be extended with the matching skills bundle and
validated for that feature.

## Validation

Before production approval, verify:

- The exact client-approved base image is used as the final `FROM` image.
- Autopilot dependency installation succeeds on the selected Python base.
- Autopilot starts as `leanusr` and responds on `/v1/health` at port `8080`.
- A representative model-provider request succeeds.
- MCP starts as its non-root user and responds to its `/mcp` initialization check.
- Both images work on every target architecture.
- The images pass the client’s vulnerability, SBOM, signing, and policy checks.
- No credentials or private keys are embedded in either image.

Example Autopilot health check:

```bash
curl --fail http://localhost:8080/v1/health
```

Example MCP initialization check:

```bash
curl --fail --show-error --silent \
  --request POST http://127.0.0.1:8090/mcp \
  --header 'Content-Type: application/json' \
  --header 'Accept: text/event-stream, application/json' \
  --data '{"jsonrpc":"2.0","id":0,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"healthcheck","version":"1.0"}}}'
```

## Upgrade Procedure

1. Select the next released Autopilot and MCP image versions.
2. Update the source image build arguments.
3. Rebuild both images using the approved client bases.
4. Run the validation checks in a non-production environment.
5. Publish new client-specific tags and update the deployment values.
