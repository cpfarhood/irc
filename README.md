# IRC Applications

Kubernetes manifests for IRC applications, deployed via Flux CD.

## Applications

- **The Lounge** - Modern web IRC client with persistent connections
- **ZNC** - IRC bouncer for persistent IRC presence

## Deployment

This repository is deployed to Kubernetes using **Flux CD** with variable substitution. Configuration variables (e.g., hostnames) are provided via ConfigMaps at deployment time.

**Important:** Manifests use Flux variable syntax (`${VARIABLE_NAME}`). Do not replace these with hardcoded values.

## Architecture

- **Kustomize-based**: Uses Kustomize for manifest organization
- **StatefulSets**: Both apps use StatefulSets with persistent volumes (4Gi each)
- **Security hardened**:
  - Run as non-root (UID 1000)
  - Seccomp profiles enabled (RuntimeDefault)
  - All capabilities dropped
  - Network policies configured
- **Resource managed**: CPU and memory limits set, including ephemeral storage
- **Health checks**: Liveness and readiness probes configured

## Local Development

### Validate manifests
```bash
# YAML linting
yamllint -c .yamllint.yaml .

# Test kustomize builds
kubectl kustomize .
kubectl kustomize ./thelounge
kubectl kustomize ./znc

# Validate schemas
kubectl kustomize . | kubeconform \
  -schema-location default \
  -schema-location 'https://raw.githubusercontent.com/datreeio/CRDs-catalog/main/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json' \
  -skip HTTPRoute \
  -ignore-missing-schemas
```

### Security scanning
```bash
# Trivy
trivy config --severity CRITICAL,HIGH --ignorefile .trivyignore .

# Checkov
checkov -d . --config-file .checkov.yaml
```

### Best practices
```bash
# Kube-score
kubectl kustomize . | kube-score score - \
  --ignore-test container-image-tag \
  --ignore-test container-security-context-readonlyrootfilesystem

# Polaris
kubectl kustomize . | polaris audit --format pretty
```

## CI/CD

Automated validation and security scanning via Gitea Actions:

### Validate Manifests
- YAML linting (yamllint)
- Kustomize build tests
- Kubernetes schema validation (kubeconform, skips HTTPRoute with variables)

### Security Scan
- **Trivy**: Vulnerability scanning with automated PR reviews
- **Checkov**: IaC security scanning with automated PR reviews
- Blocks PRs on critical findings, warns on high severity

### Best Practices
- **kube-score**: Kubernetes best practices analysis
- **Polaris**: Security and reliability audit with automated PR reviews
- **Resource analysis**: CPU/memory configuration review

All workflows run on push/PR to main branch.

## Documentation

See [CLAUDE.md](CLAUDE.md) for comprehensive development documentation.