# irc

Kubernetes manifests for IRC applications, deployed via Flux CD.

## Applications

- **The Lounge** - Modern web IRC client
- **ZNC** - IRC bouncer

## Deployment

This repository is deployed to Kubernetes using **Flux CD** with variable substitution. Configuration variables (e.g., hostnames) are provided via ConfigMaps at deployment time.

## CI/CD

Automated validation and security scanning via Gitea Actions:
- YAML linting and Kustomize validation
- Kubernetes schema validation (kubeconform)
- Security scanning (Trivy, Checkov)
- Best practices analysis (kube-score, Polaris)