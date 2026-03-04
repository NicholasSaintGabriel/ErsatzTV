# Recompile, Retest, and Deploy Your Own ErsatzTV Docker Image

This guide walks through a practical end-to-end flow to:

1. rebuild the app from source,
2. run tests,
3. build your own Docker image, and
4. push it to your own container registry.

It follows the same core build approach used by this repository's Dockerfile and CI workflows.

## 1) Prerequisites

Install the following on your build machine:

- **Git**
- **Docker** (with BuildKit/buildx support)
- **.NET 10 SDK** (for local compile/test outside Docker)

Optional but recommended:

- a Docker Hub, GHCR, or private registry account
- `docker compose` (for local runtime validation)

## 2) Clone and prepare

```bash
git clone https://github.com/ErsatzTV/ErsatzTV.git
cd ErsatzTV
```

If you are building your own fork/branch, check out that branch before continuing.

## 3) Recompile and retest locally (recommended)

These steps validate source changes before you spend time shipping images.

### Restore dependencies

```bash
dotnet restore
```

### Build

```bash
dotnet build -c Release --no-restore
```

### Run tests

```bash
dotnet test --blame-hang-timeout "2m" --no-restore --verbosity normal
```

> The test command mirrors the repository CI workflow.

## 4) Build your Docker image

The project Dockerfile is at `docker/Dockerfile` and already performs a full publish of `ErsatzTV.Scanner` and `ErsatzTV` as part of the image build.

From the repo root:

```bash
docker build \
  -f docker/Dockerfile \
  -t <your-registry>/<your-namespace>/ersatztv:<your-tag> \
  --build-arg INFO_VERSION="<your-tag-or-version>" \
  .
```

Example:

```bash
docker build -f docker/Dockerfile -t ghcr.io/acme/ersatztv:2026.03.04 --build-arg INFO_VERSION="2026.03.04" .
```

## 5) Smoke-test the container locally

Create local folders for config/transcode and run:

```bash
mkdir -p ./local-config ./local-transcode

docker run --rm \
  -p 8409:8409 \
  -e TZ=UTC \
  -e ETV_CONFIG_FOLDER=/config \
  -e ETV_TRANSCODE_FOLDER=/transcode \
  -v "$(pwd)/local-config:/config" \
  -v "$(pwd)/local-transcode:/transcode" \
  <your-registry>/<your-namespace>/ersatztv:<your-tag>
```

Then open `http://localhost:8409` and confirm the web UI loads.

## 6) Push to your registry

## 6a) Docker Hub vs GHCR pricing and private images

Short answer:

- **Both Docker Hub and GHCR have free tiers**.
- **Both support private repositories/images** so you can publish images that are not publicly pullable.

Practical guidance:

- **Docker Hub**:
  - Free accounts can host public images.
  - Private repositories are available, with limits/features depending on your plan.
  - Pull rate limits apply for unauthenticated users and differ by account type.
- **GHCR (GitHub Container Registry)**:
  - Commonly used with GitHub accounts/orgs and repositories.
  - Supports public and private container packages.
  - Private package access is controlled by GitHub permissions and tokens.

Because plans/limits can change over time, verify current pricing and limits before relying on a specific quota:

- Docker Hub pricing/limits: https://www.docker.com/pricing/
- GHCR billing: https://docs.github.com/en/billing/concepts/product-billing/github-packages

### Restricting pulls (private images)

To ensure others cannot pull your image anonymously:

1. create the repository/package as **private**,
2. only grant access to specific users/teams,
3. require authenticated pulls (`docker login`) with credentials/token that has package read permission.

If you later need limited distribution, you can keep the image private and issue read-only credentials to selected users/systems.

### Authenticate

```bash
docker login <your-registry>
```

### Push

```bash
docker push <your-registry>/<your-namespace>/ersatztv:<your-tag>
```

## 7) (Optional) Build and push multi-arch images

If you want one tag supporting multiple CPU architectures, use buildx:

```bash
docker buildx create --name ersatztv-builder --use --bootstrap

docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -f docker/Dockerfile \
  -t <your-registry>/<your-namespace>/ersatztv:<your-tag> \
  --build-arg INFO_VERSION="<your-tag-or-version>" \
  --push \
  .
```

## 8) Deploy with docker compose

You can adapt `docker/docker-compose.yml`:

- replace the `build:` block with `image: <your-registry>/<your-namespace>/ersatztv:<your-tag>`
- keep port/volume/environment settings that fit your environment

Minimal example:

```yaml
services:
  ersatztv:
    image: <your-registry>/<your-namespace>/ersatztv:<your-tag>
    ports:
      - "8409:8409"
    environment:
      TZ: UTC
    volumes:
      - ersatztv:/config
    tmpfs:
      - /transcode

volumes:
  ersatztv:
```

Run:

```bash
docker compose up -d
```

## 9) Updating your image later

For each update cycle:

1. pull latest source changes,
2. rerun restore/build/test,
3. rebuild image with a new tag,
4. push,
5. redeploy with the new tag.

Using immutable tags (for example, date or git SHA) is safer than reusing `latest`.
