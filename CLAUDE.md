# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **homelab-infra** repository, part of the Flux D2 architecture for managing Kubernetes cluster infrastructure using Flux Operator and OCI Artifacts.

**Purpose:**
- Define Kubernetes infrastructure components (cluster add-ons, monitoring, logging)
- Manage cluster-wide definitions (Namespaces, RBAC, storage classes)
- Implement pod security standards and network policies
- Managed by platform team with cluster admin privileges

**Part of D2 Architecture:**
- [homelab-fleet](https://github.com/abes140377/homelab-fleet) - Cluster fleet management
- [homelab-infra](https://github.com/abes140377/homelab-infra) - Infrastructure components (this repo)
- [homelab-apps](https://github.com/abes140377/homelab-apps) - Application delivery (planned)

## Repository Structure

```
homelab-infra/
├── .github/
│   └── workflows/           # CI/CD workflows for OCI artifacts
│       ├── push-artifact.yaml      # Push on main branch
│       ├── release-artifact.yaml   # Release on tags
│       └── image-updates.yaml      # Automated updates
├── components/              # Kubernetes add-ons with Flux HelmReleases
│   ├── cert-manager/
│   │   ├── controllers/    # HelmRelease definitions
│   │   │   ├── base/       # Common definitions
│   │   │   ├── production/ # Production-specific values
│   │   │   └── staging/    # Staging-specific values
│   │   └── configs/        # Custom Resources (Issuers, Certificates)
│   │       ├── base/
│   │       ├── production/
│   │       └── staging/
│   └── monitoring/
│       ├── controllers/    # kube-prometheus-stack, metrics-server
│       │   ├── base/
│       │   ├── production/
│       │   └── staging/
│       └── configs/        # ServiceMonitor, PodMonitor, RBAC
│           ├── base/
│           ├── production/
│           └── staging/
├── update-policies/         # Flux ImageRepository and ImagePolicy
│   ├── cert-manager.yaml
│   ├── kube-prometheus-stack.yaml
│   └── metrics-server.yaml
├── .gitignore
├── LICENSE
├── README.md
└── CLAUDE.md               # This file
```

### Component Structure Pattern

Each component follows this structure:

```
component-name/
├── controllers/             # CRD definitions and controllers
│   ├── base/               # Common definitions (Namespaces, RBAC, HelmReleases)
│   ├── production/         # Production-specific HelmRelease values
│   └── staging/            # Staging-specific HelmRelease values
└── configs/                # Custom Resources for controllers
    ├── base/               # Common CR definitions
    ├── production/         # Production-specific values
    └── staging/            # Staging-specific values
```

**Important:** Controllers are reconciled before configs to ensure CRDs are available.

## Current Components

### cert-manager
- **Purpose:** TLS certificate management
- **Controller:** jetstack/cert-manager Helm chart
- **Configs:** ClusterIssuers for Let's Encrypt (staging and production)
- **Update Policy:** Monitors `quay.io/jetstack/charts/cert-manager` with semver `>=1.0.0`

### monitoring
- **Purpose:** Metrics collection and visualization
- **Controllers:**
  - kube-prometheus-stack (Prometheus, Grafana, Alertmanager)
  - metrics-server (node and pod metrics)
- **Configs:**
  - ServiceMonitor and PodMonitor for Flux components
  - RBAC for monitoring access
  - Grafana dashboards (cluster, control-plane)
- **Update Policies:**
  - kube-prometheus-stack monitoring OCI charts
  - metrics-server monitoring OCI charts

## CI/CD Workflows

### Push Artifact (push-artifact.yaml)
- **Trigger:** Push to `main` branch or manual dispatch
- **Action:** Builds and pushes OCI artifacts to GHCR for all components
- **Matrix:** cert-manager, monitoring
- **Output:** `ghcr.io/abes140377/homelab-infra/{component}:latest`
- **Signing:** Cosign signatures for all artifacts

### Release Artifact (release-artifact.yaml)
- **Trigger:** Git tags in format `component/version` (e.g., `cert-manager/v1.0.0`)
- **Action:** Builds and pushes versioned OCI artifact for specific component
- **Output:**
  - `ghcr.io/abes140377/homelab-infra/{component}:version`
  - `ghcr.io/abes140377/homelab-infra/{component}:latest-stable`
- **Signing:** Cosign signatures

### Image Updates (image-updates.yaml)
- **Trigger:** Branch creation `image-updates` or manual dispatch
- **Action:** Creates PR from `image-updates` branch to `main`
- **Purpose:** Automate Flux image update automation workflow

## Common Workflows

### Adding a New Component

1. Create component directory structure:
```bash
mkdir -p components/{component-name}/{controllers,configs}/{base,production,staging}
```

2. Add base controller definitions:
   - Namespace
   - RBAC (if needed)
   - HelmRepository (if using Helm)
   - HelmRelease with common values

3. Add environment-specific overlays in production/staging directories

4. Create custom resources in `configs/` directories

5. Add update policy in `update-policies/{component-name}.yaml`

6. Update workflow matrix in `.github/workflows/push-artifact.yaml`

7. Test locally with Flux:
```bash
flux build kustomization components/{component-name}/controllers/base
flux build kustomization components/{component-name}/configs/base
```

### Updating Component Configuration

1. Make changes in appropriate directory (base, production, or staging)
2. Test changes:
```bash
flux build kustomization components/{component}/controllers/{env}
flux build kustomization components/{component}/configs/{env}
```
3. Commit and push to trigger OCI artifact build
4. Fleet repository will pick up new artifact automatically

### Creating a Release

1. Ensure changes are merged to `main` and artifact is built
2. Create and push tag:
```bash
git tag {component-name}/v{version}
git push origin {component-name}/v{version}
```
3. Release workflow builds versioned OCI artifact
4. Update fleet repository to use new version

## Flux Integration

This repository is consumed by the fleet repository via OCI artifacts:

```yaml
# In fleet repository
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: OCIRepository
metadata:
  name: homelab-infra-cert-manager
spec:
  url: oci://ghcr.io/abes140377/homelab-infra/cert-manager
  ref:
    tag: latest
```

The fleet repository creates ResourceSets that reference these OCI artifacts, which are then deployed to target clusters.

## Best Practices

### Component Design
- Keep base configuration minimal and environment-agnostic
- Use Kustomize overlays for environment-specific customization
- Document all custom values in HelmRelease comments
- Always test with `flux build` before committing

### Security
- All artifacts are signed with Cosign
- Secrets should be managed via SOPS in fleet repository, not here
- RBAC should follow principle of least privilege
- Review security contexts and pod security standards

### Versioning
- Use semantic versioning for releases
- Tag format: `{component}/v{major}.{minor}.{patch}`
- `:latest` for continuous deployment (main branch)
- `:latest-stable` for last versioned release
- Document breaking changes in release notes

### Testing
- Validate manifests with `flux build kustomization`
- Test in staging environment before production
- Use Flux drift detection to verify deployments
- Monitor Flux reconciliation status

## Troubleshooting

### Component Not Updating
1. Check OCI artifact was pushed:
```bash
flux reconcile source oci homelab-infra-{component} --with-source
```

2. Verify artifact exists in GHCR:
```bash
crane ls ghcr.io/abes140377/homelab-infra/{component}
```

3. Check ResourceSet status in fleet repository

### HelmRelease Failing
1. Check HelmRelease status:
```bash
flux get helmreleases -n {namespace}
```

2. Review Helm values:
```bash
flux get helmrelease {name} -n {namespace} -o yaml
```

3. Check for CRD issues (controllers must be ready before configs)

### Image Update Automation Not Working
1. Verify ImageRepository is fetching:
```bash
flux get image repository {component}
```

2. Check ImagePolicy evaluation:
```bash
flux get image policy {component}
```

3. Review update-policies definitions

## Development Environment

### Required Tools
- `flux` CLI (v2.5+)
- `kubectl` with cluster access
- `crane` or `docker` for OCI image operations
- `cosign` for signature verification
- `kustomize` (optional, bundled with kubectl)

### Local Testing
```bash
# Build and validate component manifests
flux build kustomization components/{component}/controllers/base

# Test with diff against cluster
flux diff kustomization {name} --path components/{component}/controllers/{env}

# Push to OCI (requires GHCR authentication)
flux push artifact oci://ghcr.io/abes140377/homelab-infra/{component}:test \
  --path=./components/{component} \
  --source=$GITHUB_SERVER_URL/$GITHUB_REPOSITORY \
  --revision=$GITHUB_REF_NAME@sha1:$GITHUB_SHA
```

## References

- [Flux D2 Architecture Guide](https://raw.githubusercontent.com/abes140377/distribution/main/guides/ControlPlane_Flux_D2_Reference_Architecture_Guide.pdf)
- [Flux Operator Documentation](https://fluxcd.control-plane.io/operator/)
- [Fleet Repository](https://github.com/abes140377/homelab-fleet)
- [Flux Image Automation](https://fluxcd.io/flux/guides/image-update/)

## Important Notes

- This repository is reconciled with **cluster admin** privileges
- Access is restricted to platform team members
- Never commit secrets - use SOPS in fleet repository
- Controllers must be ready before configs are applied
- Always test in staging before production
- OCI artifacts are automatically built on push to main
