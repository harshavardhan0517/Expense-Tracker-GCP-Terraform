# Expense Tracker – GCP Infrastructure & CI/CD Automation

A three-tier MERN (MongoDB, Express.js, React, Node.js) expense tracking application, paired with fully automated Google Cloud infrastructure provisioning and Azure DevOps CI/CD pipelines. The project demonstrates end-to-end infrastructure-as-code and keyless, OIDC-based deployment automation from Azure DevOps to GCP.

## What It Does

The application is a proof-of-concept expense tracker supporting:

- User registration
- Creating, viewing, updating, and deleting expenses (full CRUD)

The focus of this project is less on the application itself and more on how it's **provisioned, built, and deployed** — repeatable infrastructure and a secure, automated release pipeline.

## Repository Structure

```
.
├── app/          # MERN application source code (frontend + backend)
└── terraform/    # Terraform IaC for the GCP environment + infra pipeline
```

## Architecture

**Application (3-tier, single VM):**

```
React (frontend, static build)  →  Express/Node.js (backend API)  →  MongoDB (self-hosted on the same VM)
```

- Frontend is built to a static `dist/` bundle.
- Backend is packaged as a `tar.gz` build artifact.
- MongoDB runs self-hosted directly on the Compute Engine VM (no Atlas, no managed DB service).
- Artifacts are **not containerized** — they're deployed as plain build artifacts (static files + backend archive) directly onto the VM, not via Docker.

**Infrastructure (GCP, via Terraform):**

- VPC networking and subnets
- Firewall rules
- Compute Engine VM (application host)
- Cloud NAT (outbound connectivity for the private VM)
- Artifact Registry (stores versioned build artifacts)
- Secret Manager (application/environment secrets)
- IAM (least-privilege service accounts and bindings)
- Cloud Storage (supporting storage needs, e.g. Terraform state or build artifacts)

## CI/CD

CI/CD is implemented in **Azure DevOps**, with two separate pipelines:

| Pipeline | Responsibility |
|---|---|
| **Infrastructure pipeline** | Runs Terraform (plan/apply) to provision and update the GCP environment |
| **Application pipeline** | Builds the frontend (`dist/`) and backend (`.tar.gz`), pushes versioned artifacts to Google Artifact Registry, and deploys the latest build to the target VM |

### Authentication: OIDC + Workload Identity Federation

Instead of long-lived GCP service-account keys stored as pipeline secrets, authentication from Azure DevOps to GCP is established using:

- **Microsoft Entra ID** as the OIDC token issuer, via a dedicated App Registration / service principal created for the pipeline — not a personal user identity
- **GCP Workload Identity Federation (WIF)** to exchange the Entra ID OIDC token for short-lived GCP credentials

This means the pipelines authenticate to GCP without any static, long-lived key material ever being stored in Azure DevOps, and without relying on any individual's personal credentials.

### Deployment Flow

1. Application pipeline builds the frontend and backend.
2. Build artifacts (`frontend dist` + `backend tar.gz`) are versioned and pushed to **Google Artifact Registry**.
3. The same pipeline run deploys the latest build in parallel to the target Compute Engine VM, replacing the running version.

### Manual Stage Approval

Both pipelines use **manual triggers on each stage** rather than fully automatic progression — a run must be explicitly approved to move from one stage to the next (e.g. build → deploy, or plan → apply). This adds a deliberate checkpoint before changes reach infrastructure or the running application.

## Terraform State & Safety

- Terraform state is managed remotely to support consistent, team-safe plan/apply cycles.
- Lifecycle safeguards (e.g. `prevent_destroy`, careful resource targeting) are used on stateful resources to reduce the risk of accidental data loss or unintended resource destruction during infrastructure changes — particularly important given MongoDB is self-hosted on the VM rather than a managed, backed-up database service.
- The infrastructure pipeline exposes a **second, explicit parameter for "safe destroy"**: destroy operations only run against resources holding persistent data (e.g. the VM/MongoDB disk) when this parameter is deliberately set, separate from the normal plan/apply trigger. This prevents a routine pipeline run from accidentally tearing down persistent state.

## Getting Started

### Prerequisites

- Google Cloud project with billing enabled
- Terraform CLI installed
- Azure DevOps organization/project with pipeline permissions
- Microsoft Entra ID app registration configured for Workload Identity Federation with GCP
- Node.js (for local app development)

### Provisioning Infrastructure

```bash
cd terraform
terraform init
terraform plan
terraform apply
```

This provisions the VPC, subnets, firewall rules, the Compute Engine VM, Cloud NAT, Artifact Registry, Secret Manager entries, IAM bindings, and Cloud Storage buckets required to run the application.

### Running the App Locally

```bash
cd app
# see app/README or package.json scripts for frontend/backend-specific setup
```

Local development expects a running MongoDB instance (matching the self-hosted setup used in production) and any required environment variables for connecting the backend to it.

### Deploying

Deployment is handled by the Azure DevOps application pipeline, which:

1. Builds the frontend and backend
2. Publishes versioned artifacts to Artifact Registry
3. Deploys the latest build to the VM

Manual/local deployment isn't the primary workflow for this project — the pipeline is the source of truth for releases.

## Key Design Decisions

- **No Docker on the app VM** — build artifacts (static frontend files, backend `tar.gz`) are deployed directly, keeping the runtime environment simple and avoiding container-orchestration overhead for a single-VM PoC.
- **Self-hosted MongoDB** — chosen over Atlas/Firestore to keep the full stack within the provisioned VM and demonstrate self-managed database operations as part of the infrastructure story.
- **OIDC + Workload Identity Federation** — eliminates long-lived GCP service-account keys from the CI/CD system, reducing credential-leakage risk.
- **Two separate pipelines** (infra vs. app) — decouples infrequent infrastructure changes from frequent application releases, reducing blast radius of each pipeline run.

## Roadmap / Possible Improvements

- Containerize the application for easier scaling beyond a single VM
- Move MongoDB to a managed or replicated setup for durability
- Add automated rollback on failed deployments
- Expand Terraform modules for multi-environment (dev/staging/prod) support

## License

No license has been applied to this repository yet.