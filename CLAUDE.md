# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains Kubernetes manifests for deploying IRC-related applications (The Lounge web client and ZNC bouncer) using Kustomize. The infrastructure is deployed to a Kubernetes cluster with Flux CD and uses Gitea Actions for CI/CD validation and security scanning.

## Architecture

### Kustomize Structure
- **Root kustomization.yaml**: Aggregates all application components (thelounge, znc)
- **Application directories**: Each contains its own kustomization.yaml with associated manifests:
  - `thelounge/`: Web-based IRC client (StatefulSet, Service, HTTPRoute, NetworkPolicy)
  - `znc/`: IRC bouncer (StatefulSet, Service, NetworkPolicy)
- **Commented applications**: bitlbee and inspircd are mentioned but not currently deployed

### Application Components
Both applications follow the same pattern:
- **StatefulSet**: Deploys the main container with persistent storage via volumeClaimTemplates (4Gi)
- **Service**: Exposes the application (thelounge: 9000, znc: 6501)
- **NetworkPolicy**: Controls network ingress/egress
- **HTTPRoute**: (thelounge only) Gateway API routing configuration

### Resource Configuration
- Priority class: `low-priority` for both applications
- Resource requests/limits: 100m/500m CPU, 256Mi/512Mi memory
- Security: `automountServiceAccountToken: false`, `allowPrivilegeEscalation: false`
- Probes: Both liveness and readiness probes configured for reliability

### Polaris Exemptions
All manifests have Polaris exemptions:
- `runAsRootAllowed-exempt`: Containers need root for their base images
- `tagNotSpecified-exempt`: Using `latest` tags
- `topologySpreadConstraint-exempt`: Single-replica deployments don't need spread constraints

## CI/CD Pipeline

### Gitea Actions Workflows
Located in `.gitea/workflows/`, three main workflows run on push/PR to main:

1. **validate.yaml** - Manifest validation:
   - YAML linting with yamllint
   - Kustomize build tests (root + individual apps)
   - Kubernetes schema validation with kubeconform
   - Flux build validation

2. **security.yaml** - Security scanning with PR review automation:
   - **Trivy**: Scans for vulnerabilities, posts PR reviews
   - **Checkov**: IaC security scanning, posts PR reviews
   - PR review states: `REQUEST_CHANGES` (critical), `COMMENT` (high), `APPROVED` (clean)
   - Only scans changed YAML/YML/TF files in PRs

3. **best-practices.yaml** - Kubernetes best practices:
   - **kube-score**: Best practices analysis
   - **Polaris**: Security and reliability audit with PR reviews
   - Resource usage analysis
   - Polaris enforces minimum 70% score and blocks on dangers

### PR Review System
Security and best practices workflows automatically review PRs:
- **Trivy/Checkov**: Critical findings block, high findings warn
- **Polaris**: Danger findings block, warnings comment
- Reviews posted via Gitea API with detailed tables
- Requires tokens: `TRIVY_GITEA_TOKEN`, `CHECKOV_GITEA_TOKEN`, `POLARIS_GITEA_TOKEN`

## Development Commands

### Local Validation
```bash
# YAML linting
yamllint -c .yamllint.yaml .

# Build and validate root kustomization
kubectl kustomize . > /tmp/manifests.yaml

# Build individual app kustomizations
kubectl kustomize ./thelounge
kubectl kustomize ./znc

# Validate with kubeconform
kubectl kustomize . | kubeconform \
  -schema-location default \
  -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' \
  -summary -ignore-missing-schemas

# Flux validation
flux build kustomization irc --path . --dry-run
```

### Security Scanning
```bash
# Run Trivy config scan
trivy config --severity CRITICAL,HIGH --ignorefile .trivyignore .

# Run Checkov scan
checkov -d . --config-file .checkov.yaml --output cli
```

### Best Practices Analysis
```bash
# Run kube-score
kubectl kustomize . | kube-score score - \
  --ignore-test pod-networkpolicy \
  --ignore-test deployment-has-poddisruptionbudget \
  --ignore-test container-security-context-user-group-id \
  --ignore-test container-security-context-readonlyrootfilesystem

# Run Polaris audit
kubectl kustomize . | polaris audit --format pretty --set-exit-code-on-danger --set-exit-code-below-score 70
```

## Configuration Files

### Security and Validation
- `.yamllint.yaml`: YAML linting rules (line-length, document-start, truthy disabled)
- `.checkov.yaml`: Checkov configuration, skips CKV_K8S_21 (namespace) and CKV_K8S_43 (image tags)
- `.trivyignore`: Ignores CVE-2021-26720 (Avahi) and CVE-2023-52425 (accepted risks)
- `configmap.yaml.example`: Template for hostname configuration (not tracked in repo)

## Key Patterns

### Adding New Applications
1. Create app directory with kustomization.yaml
2. Add required manifests (statefulset, service, networkpolicy)
3. Reference in root kustomization.yaml resources
4. Include Polaris exemptions if needed
5. Set priorityClassName: low-priority
6. Disable automountServiceAccountToken
7. Configure resource requests/limits and probes

### Modifying Workflows
- All workflows use `catthehacker/ubuntu:act-latest` container for act compatibility
- PR review jobs require fetch-depth: 0 for git diff operations
- Security tokens should use dedicated secrets (not shared GITEA_TOKEN)
- Exit code 1 on critical findings, 0 on warnings/pass

## Commit Message Format
When making commits, include credits:
```
<main commit message>

Generated with [Claude Code](https://claude.ai/code)
via [Happy](https://happy.engineering)

Co-Authored-By: Claude <noreply@anthropic.com>
Co-Authored-By: Happy <yesreply@happy.engineering>
```
