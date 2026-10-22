# coder-k3s-homelab

A single-node k3s cluster on my Fedora homelab server running [Coder](https://github.com/coder/coder) (open source), plus notes on what broke and how I diagnosed it.

This is a homelab project to learn Kubernetes and Terraform hands-on. It is one node, not production, and I did it with Claude as a guide: the commands, outputs, and failures in these notes are from my own terminal.

## What is here

| Path | What it is |
|---|---|
| `k3s/config.yaml` | k3s config that disables the bundled Traefik and ServiceLB |
| `helm/coder-values.yaml` | Helm values for Coder (NodePort service, Postgres via secret) |
| `template/` | The Coder workspace template (Terraform) and what I changed in it |
| `docs/setup.md` | The install steps in order, with the Fedora specifics |
| `docs/troubleshooting.md` | Runbook of the failures I hit and the evidence that identified each |

## What I built

1. k3s on Fedora (SELinux enforcing, firewalld on), alongside 30+ existing Podman containers
2. Postgres in-cluster (Bitnami chart) with a secret holding the connection URL
3. Coder installed with Helm, exposed on a NodePort
4. The Kubernetes starter template with the `code-server` module, plus one edit: a `coder_parameter` that becomes an environment variable inside the workspace

## What I would not claim

- One node, no HA, no ingress or TLS
- I edited a starter template. I did not write one from scratch.
- Postgres uses a legacy Bitnami image (the chart's default images are no longer freely published), which is fine for a lab and not for production.

## Reproduce

See `docs/setup.md`. Pin your own chart version, and generate your own database password.
