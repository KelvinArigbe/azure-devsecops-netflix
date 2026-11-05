![GitHub Actions](https://github.com/KelvinArigbe/azure-devsecops-netflix/actions/workflows/devsecops.yml/badge.svg?branch=main)

<div align="center">

  <img src="./public/Azure-DevSecOps-Architecture.png" alt="Azure DevSecOps Project Architecture" width="100%">

  **Build | Secure | Deliver | Deploy | Monitor**

  *An Azure-focused DevSecOps engineering project*

</div>

# Azure DevSecOps Netflix-Style Application

A hands-on DevSecOps project that takes a React and TypeScript application through containerisation, image vulnerability scanning, Azure Container Registry (ACR), and automated deployment to an Azure Linux virtual machine.

The application is served by Nginx in Docker. GitHub Actions runs the build and security gate, while a self-hosted runner on the Azure VM performs the Azure deployment using the VM's system-assigned managed identity.

## Current project status

<details>
<summary><strong>Implemented and verified</strong></summary>

- React/TypeScript application builds successfully with Node.js 22.
- Hardened container image is built from `Dockerfile.secure`.
- Trivy image scanning runs in GitHub Actions and fails the job when HIGH or CRITICAL findings are detected.
- GitHub Actions deploys from the `main` branch through a self-hosted runner on the Azure VM.
- The VM authenticates to Azure using its system-assigned managed identity.
- The image is pushed to Azure Container Registry with the Git commit SHA as its tag.
- A candidate container is started and checked over HTTP before production is replaced.
- The production container is configured to restart automatically and the deployment script includes a rollback path.
- The deployed production container was verified on the VM with an HTTP `200 OK` response.

</details>

**Not implemented as part of the active pipeline yet:** Terraform provisioning, AKS deployment, runtime monitoring, and promotion of the exact image digest that passed the hosted-runner scan. See [Known limitations and next improvements](#known-limitations-and-next-improvements).

## Architecture and delivery flow

### Pull request and build/security gate

1. A push or pull request targeting `main` triggers the GitHub Actions workflow.
2. GitHub-hosted Ubuntu runner checks out the repository and sets up Node.js 22.
3. Yarn installs dependencies using the lockfile.
4. The application is built.
5. Docker builds an image using `Dockerfile.secure`.
6. Trivy scans that image. HIGH and CRITICAL findings fail the job.

### Deployment after a push to main

1. The deployment job runs only after the build/security job passes and only for a push to `main`.
2. The self-hosted GitHub Actions runner on the Azure VM authenticates with `az login --identity`.
3. Azure CLI authenticates Docker to ACR.
4. The VM builds and pushes a commit-SHA-tagged image to ACR.
5. A candidate container starts on host port `8084` and must return a successful HTTP response.
6. The deployment script replaces `netflix-production` on host port `8082`.
7. If the production HTTP check fails, the script attempts to restore the previous image.

### Main components

| Area | Technology / implementation |
|---|---|
| Cloud platform | Microsoft Azure |
| VM operating system | Ubuntu Linux |
| Application | React and TypeScript |
| Build tool / package manager | Vite / Yarn |
| Container runtime | Docker |
| Container web server | Nginx |
| Container registry | Azure Container Registry |
| CI/CD | GitHub Actions |
| Deployment runner | GitHub self-hosted runner on Azure VM |
| Azure authentication | System-assigned managed identity |
| Registry authorization | Azure RBAC; `AcrPush` assigned to the VM identity |
| Image vulnerability scanning | Trivy |
| Image tags | Git commit SHA |

## Repository layout

| Path | Purpose |
|---|---|
| `.github/workflows/devsecops.yml` | Active GitHub Actions build, scan, and deployment workflow |
| `Dockerfile.secure` | Dockerfile used by the active workflow |
| `Dockerfile` | Original/legacy Docker build definition; not used by the active workflow |
| `.env.example` | Template for local application configuration |
| `.dockerignore` | Files excluded from Docker build context |
| `public/Azure-DevSecOps-Architecture.png` | Architecture illustration |
| `Kubernetes/` | Earlier Kubernetes manifests; not used by the current VM deployment |
| `pipeline.txt` | Legacy Jenkins pipeline reference; not used by GitHub Actions |

## Run the application locally

### Requirements

- Node.js 22
- Yarn
- Docker (for container builds)
- A TMDB API key for the application's live catalogue data

### Run with Vite

1. Clone the repository and enter its directory.
2. Install dependencies:

   ```bash
   yarn install --frozen-lockfile
   ```

3. Create a local environment file from the template:

   ```bash
   cp .env.example .env
   ```

4. Set the values in `.env`:

   ```dotenv
   VITE_APP_API_ENDPOINT_URL=https://api.themoviedb.org/3
   VITE_APP_TMDB_V3_API_KEY=your_tmdb_api_key
   ```

5. Start the development server:

   ```bash
   yarn dev
   ```

### Build and run the secure container locally

The secure Dockerfile accepts the TMDB key as a build argument:

```bash
docker build \
  --build-arg TMDB_V3_API_KEY=your_tmdb_api_key \
  -f Dockerfile.secure \
  -t netflix:secure .
```

Run the image:

```bash
docker run --rm -d --name netflix-local -p 8082:80 netflix:secure
curl -I http://localhost:8082
```

Stop it when finished:

```bash
docker rm -f netflix-local
```

## Configure the Azure deployment

The active workflow expects the following environment to exist before deployment:

1. An Azure Linux VM with Docker and Azure CLI installed.
2. A system-assigned managed identity enabled on the VM.
3. The VM identity granted the `AcrPush` role on the target ACR.
4. A self-hosted GitHub Actions runner registered to this repository, with the `linux` and `x64` labels.
5. The runner service running on the VM.
6. The ACR name and login server in the workflow's `env` section updated for the target environment.

The runner is installed in the VM user's home directory (outside the application repository). This is intentional: the application declares `"type": "module"` in its `package.json`, and keeping the runner outside that package scope avoids Node.js treating the runner's CommonJS service file as an ES module.

## Configuration and API-key handling

The application reads `VITE_APP_API_ENDPOINT_URL` and `VITE_APP_TMDB_V3_API_KEY` through Vite environment variables.

**Important:** Vite exposes variables prefixed with `VITE_` in the browser bundle. The TMDB key is therefore visible to users when the application runs. Do not use this mechanism for private credentials. Restrict the key with the provider's available controls and monitor its usage. Do not commit a real `.env` file.

The current GitHub Actions workflow does not pass a TMDB build argument to either Docker build. The image can still build, but application features that call TMDB may need the key supplied at build time. Add the key to the workflow build commands only after configuring the corresponding repository secret, and remember that a client-side key remains public in the resulting browser assets.

## Security controls demonstrated

- Separate hardened Dockerfile using Node.js 22 for the build stage.
- Nginx-based runtime image.
- Trivy container-image scan with HIGH/CRITICAL severity gating.
- GitHub Actions deployment restricted to pushes to `main`.
- Azure managed identity rather than a stored Azure service-principal password for VM-to-Azure authentication.
- ACR authorization through Azure RBAC.
- Commit-SHA image tags instead of relying only on a mutable `latest` tag.
- Candidate HTTP check before production replacement.
- Rollback attempt if the production HTTP check fails.

## Known limitations and next improvements

1. **Promote the scanned artifact.** The hosted build job creates and scans an image, but the deployment job currently builds the image again on the Azure VM. The two images are built from the same commit but are not guaranteed to be byte-for-byte identical. A stronger pipeline should publish the candidate once, scan that exact image, and deploy the same immutable digest.
2. **Supply the TMDB build argument.** The current workflow does not pass `TMDB_V3_API_KEY` into the Docker builds, so TMDB-backed application features may not work in the CI-built image until this is configured.
3. **Make deployment health checks stronger.** The current check verifies an HTTP response from Nginx; it does not validate all application features or external API connectivity.
4. **Review legacy assets before reuse.** The Kubernetes manifests still reference an older Docker Hub image and are not wired into this pipeline. The root `Dockerfile` uses an older Node.js base image. `pipeline.txt` is a legacy Jenkins reference, not the active CI/CD implementation. Update or remove these legacy files before using them.
5. **Expand security coverage.** Add dependency and secret scanning, SBOM generation, image signing, and runtime monitoring as separate improvements.
6. **Add Infrastructure as Code.** Provision the VM, networking, ACR, managed identity, and role assignments reproducibly with Terraform or Bicep.

## Application screenshots

The following screenshots show the React application used for this engineering project.

<div align="center">
  <img src="./public/assets/home-page.png" alt="Application home page" width="90%">
  <p>Application home page</p>

  <img src="./public/assets/grid-genre.png" alt="Genre grid" width="90%">
  <p>Genre grid</p>

  <img src="./public/assets/detail-modal.png" alt="Movie details" width="90%">
  <p>Movie details</p>

  <img src="./public/assets/watch.png" alt="Video player" width="90%">
  <p>Video player</p>
</div>

---

This repository documents a working Azure VM-based DevSecOps deployment implementation and the next engineering improvements required to make the delivery process more reproducible and production-grade.
