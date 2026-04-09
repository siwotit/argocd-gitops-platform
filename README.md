# ArgoCD — GitOps Platform on AWS EKS

## Overview

Production GitOps platform built on ArgoCD, managing the full infrastructure and application lifecycle on AWS EKS. Uses the App-of-Apps pattern with ApplicationSets to declaratively manage all workloads across multiple Git repositories — from cluster infrastructure (Helm charts) to application deployments and image updates.

## Architecture

```
                    ┌─────────────────────────────┐
                    │   Bootstrap (root-app.yaml)  │
                    │   Applied once via kubectl   │
                    └──────────────┬──────────────┘
                                   │ syncs
                    ┌──────────────▼──────────────┐
                    │   argocd-appsets (App)       │
                    │   watches: gitops-argocd/    │
                    │           applicationsets/   │
                    └──┬──────────┬───────────┬───┘
                       │          │           │
           ┌───────────▼─┐  ┌─────▼──────┐  ┌▼────────────────┐
           │ infra-helm  │  │infra-mfsts │  │ apps / payouts  │
           │ ApplicationSet│  │ApplicationSet│  │ ApplicationSets │
           └───────────┬─┘  └─────┬──────┘  └┬────────────────┘
                       │          │           │
              Helm charts    Raw manifests   Kustomize overlays
           (gitops-infra)  (gitops-infra)  (gitops-apps / gitops-payouts)
```

## Key Design Decisions

- App-of-Apps bootstrap — a single `root-app.yaml` applied once seeds the entire platform. ArgoCD self-manages from that point.
- Separate repos per concern — `gitops-argocd`, `gitops-infra`, `gitops-apps`, `gitops-payouts` — clean separation of platform, infrastructure, and application concerns.
- Helm multi-source — infrastructure ApplicationSet uses two sources per app: the upstream Helm chart repo + the values repo (`gitops-infra`), keeping chart versions and config independently versioned.
- ArgoCD Image Updater — watches ECR for new image tags matching a regex pattern and writes back to Git automatically, closing the CI→CD loop without manual intervention.
- HA install — ArgoCD deployed in high-availability mode via Kustomize, with a patched Redis image from a private ECR registry.
- Gateway API exposure — ArgoCD server exposed via HTTPRoute through the Istio Gateway, no classic Ingress.

## What Gets Managed

| ApplicationSet | Type | Manages |
|---|---|---|
| `infrastructure-helm` | Helm multi-source | Istio, AWS LBC, ExternalDNS, Cert-Manager, Karpenter, KEDA, Prometheus, GitLab, Argo Rollouts, Image Updater |
| `infrastructure-manifests` | Kustomize | CRDs, HTTPRoutes, NodePools, ClusterIssuers, StorageClass, TriggerAuth |
| `apps` | Kustomize | jsapp (dev) |
| `payouts-apps` | Kustomize + Image Updater | pyapp (dev) — auto image updates from ECR |

## Tech Stack

- ArgoCD v2 (HA)
- ArgoCD Image Updater
- Argo Rollouts
- ApplicationSet controller
- Kustomize
- Helm 3
- AWS EKS
- AWS ECR
- Istio Gateway API (HTTPRoute for ArgoCD UI)
- Multi-repo GitOps structure

## Project Structure

```
.
├── README.md
├── bootstrap/
│   └── root-app.yaml                   # single apply to seed the platform
├── argocd/
│   ├── kustomization.yaml              # HA install + patches
│   ├── httproute.yaml                  # ArgoCD UI via Gateway API
│   ├── argocd-cm-patch.yaml            # custom resource ignore rules
│   └── argocd-server-patch.yaml        # server config patch
└── applicationsets/
    ├── apps-appset.yaml                # application workloads
    ├── payouts-apps-appset.yaml        # payouts workloads + image updater
    ├── infrastructure-helm-appset.yaml # all infra Helm charts
    └── infrastructure-manifests-appset.yaml  # raw infra manifests
```
