
# Workspace configure, run, and build

This guide covers how to configure the workspace, build images locally, and build or push images through GitHub Actions.

## Prerequisites

- Docker Desktop or Docker Engine with Buildx enabled.
- GitHub CLI (`gh`) installed and authenticated.
- A GitHub personal access token (PAT) with `write:packages` if you want to push to GHCR locally.

## 1. Configure the workspace

### 1.1 Clone the repository

```bash
git clone https://github.com/aks-tech-lab/oz-dev-environments.git
cd oz-dev-environments
```

### 1.2 Verify Docker and Buildx

```bash
docker version
docker buildx version
```

If Buildx is not available, enable it in Docker Desktop settings or install the `docker-buildx` plugin for your Docker engine.

### 1.3 Optional: prepare GHCR credentials for local pushes

If you plan to push from your local machine, log in to GHCR with a PAT:

```bash
echo "<PAT>" | docker login ghcr.io -u <github-username> --password-stdin
```

Notes:
- The PAT must include `write:packages`.
- Use a separate token from CI secrets.

## 2. Build locally (no push)

This builds the image on your machine and loads it into the local Docker engine.

### 2.1 Build a single tag (example: DuckOS)

```bash
docker buildx build --platform linux/amd64 --load \
	-t ghcr.io/aks-tech-lab/oz-dev-environments:DuckOS \
	.
```

### 2.2 Build multiple tags locally

```bash
for tag in DuckOS latest pre-build release; do
	docker buildx build --platform linux/amd64 --load \
		-t ghcr.io/aks-tech-lab/oz-dev-environments:${tag} \
		.
done
```

### 2.3 Run a local image

```bash
docker run --rm -it ghcr.io/aks-tech-lab/oz-dev-environments:DuckOS bash
```

## 3. Build and push locally to GHCR

This builds multi-arch images and pushes them to GHCR.

### 3.1 Build and push a single tag

```bash
docker buildx build --platform linux/amd64,linux/arm64 --push \
	-t ghcr.io/aks-tech-lab/oz-dev-environments:DuckOS \
	.
```

### 3.2 Build and push multiple tags

```bash
for tag in DuckOS latest pre-build release; do
	docker buildx build --platform linux/amd64,linux/arm64 --push \
		-t ghcr.io/aks-tech-lab/oz-dev-environments:${tag} \
		.
done
```

## 4. Build and push with GitHub Actions

The workflow builds and pushes all tags defined in the matrix.

### 4.1 Configure GitHub Actions secrets

Add the PAT to the repository secrets as:

- `GH_CODESPACE_PAT_KEY`

The PAT must include `write:packages`.

### 4.2 Trigger the workflow from the CLI

```bash
gh workflow run build-images.yml --ref MASTER -R aks-tech-lab/oz-dev-environments
```

### 4.3 Check workflow status

```bash
gh run list -R aks-tech-lab/oz-dev-environments
gh run watch -R aks-tech-lab/oz-dev-environments
```

## 5. Customize build features

The Dockerfile supports optional language and tooling features via build args:

- `INSTALL_RUST`
- `INSTALL_GO`
- `INSTALL_JAVA`
- `INSTALL_DOTNET`
- `INSTALL_RUBY`
- `INSTALL_BROWSERS`
- `INSTALL_CODING_AGENTS`

Example with Go enabled:

```bash
docker buildx build --platform linux/amd64 --load \
	--build-arg INSTALL_GO=true \
	--build-arg GO_VERSION=1.23.4 \
	-t ghcr.io/aks-tech-lab/oz-dev-environments:go-enabled \
	.
```

## 6. Verify the image

```bash
docker run --rm ghcr.io/aks-tech-lab/oz-dev-environments:DuckOS node --version
docker run --rm ghcr.io/aks-tech-lab/oz-dev-environments:DuckOS python --version
```

## 7. Use the image with Oz and Warp

Create an Oz environment pointing to a tag you published:

```bash
oz environment create \
	--name my-oz-env \
	--docker-image ghcr.io/aks-tech-lab/oz-dev-environments:DuckOS \
	--repo owner/repo \
	--setup-command "echo ready"
```

Run an agent against that environment:

```bash
oz agent run-cloud --environment my-oz-env --prompt "Run build and report results"
```

## 8. Align branches and tags with releases

The workflow is set to run on pushes to `MASTER` and also supports manual runs via
`workflow_dispatch`.

Current image tags:

- DuckOS
- latest
- pre-build
- release

If you follow semantic versioning, add versioned tags (for example, `v1.2.0`) to the matrix or
create a release workflow that tags images based on GitHub Releases.

## 9. GHCR cleanup and retention

To keep GHCR storage under control, you can set a retention policy in the GitHub Packages UI or
use `gh` to delete old package versions.

List package versions:

```bash
gh api \
	-H "Accept: application/vnd.github+json" \
	/user/packages/container/oz-dev-environments/versions
```

Delete a specific version by ID:

```bash
gh api \
	-X DELETE \
	-H "Accept: application/vnd.github+json" \
	/user/packages/container/oz-dev-environments/versions/<version-id>
```

## Troubleshooting

- If `gh workflow run` fails, confirm you are authenticated: `gh auth status`.
- If builds are slow, ensure Buildx and QEMU are set up for multi-arch builds.
- If a push fails, verify the PAT scope and GHCR login status.
