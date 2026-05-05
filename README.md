# Home Cluster

Where the majority of my home services reside. Runs on 3 Talos VMs on a single TrueNAS Scale machine. All persistent storage is NFS-backed by the same machine - keeping the cluster stateless and simple to rebuild.

## Hardware

Each Talos VM has 12GiB of RAM and 2 cores of my i7 8700k allocated. Each of the 3 nodes is a control-plane - a stacked etcd topology where nodes are both control-planes and workers.

## Stack

|               | Workloads                                           |
|---------------|-----------------------------------------------------|
| Core          | Talos, Cilium, CoreDNS, Flux CD                     |
| Ingress       | Traefik, Gateway API, Middlewares                   |
| DNS/Proxy     | Cloudflare, Cloudflared, External-DNS, cert-manager |
| Storage       | OpenEBS, NFS CSI, Crunchy Postgres                  |
| Observability | Prometheus stack, Goldilocks                        |
| Security      | SOPS, OpenBao, default-deny CNPs                    |
| Automation    | Renovate, pre-commit, flux-diff                     |

## Repository Structure

```
kubernetes/
├── apps/          # Workloads organized by namespace
├── bootstrap/     # Talos machine configs and initial Helm installs (Cilium, Flux, etc.)
├── components/    # Shared kustomize components (default-deny and allow-dns CNPs)
└── flux/
    ├── cluster/   # Root Flux kustomization — entrypoint for reconciliation
    └── meta/      # Helm/Git/OCI repository sources and cluster-wide settings
```

## GitOps Workflow

Changes are made to a feature-branch, then a PR from feature-branch -> main tests schema with flux-diff. After merging, the GitHub webhook fires which notifies the flux notification-controller, beginning the reconciliation process.

## Security

No router ports are open - all external connections are proxied via Cloudflare. Connections flow through the CF tunnel to the cloudflared pod, then to a traefik-external pod which only proxies the public-facing services. Internal-only services are handled by a separate traefik instance and gateway CR, which also follows a more strict local-only IP whitelist enforced by Traefik middlewares.

Precise CiliumNetworkPolicies have been created for the entire cluster based on Cilium's Hubble observations and known connections. A default-deny and allow-dns policy was established namespace-by-namespace via kustomize components simultaneous to per-app CNP application. The CNPs' primary purpose is to prevent an attacker with a compromised pod from jumping to or compromising other pods, services, or machines.

## Acknowledgments

- Originally bootstrapped from [onedr0p/cluster-template](https://github.com/onedr0p/cluster-template) which served as a great starting point.
- Special thanks to Kevin for getting me into containerization and k8s.
