# Kubernetes DevSecOps Platform (Kustomize)

![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)
![Kustomize](https://img.shields.io/badge/kustomize-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)
![Falco](https://img.shields.io/badge/falco-%2300AEC7.svg?style=for-the-badge&logo=falco&logoColor=white)
![Trivy](https://img.shields.io/badge/trivy-%231904DA.svg?style=for-the-badge&logo=aqua&logoColor=white)
![Prometheus](https://img.shields.io/badge/prometheus-%23E6522C.svg?style=for-the-badge&logo=prometheus&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/github%20actions-%232671E5.svg?style=for-the-badge&logo=githubactions&logoColor=white)

A security-hardened Kubernetes deployment of a multi-tier application, built with Kustomize overlays and a defense-in-depth posture. The project layers preventive controls (image scanning, pod hardening, admission enforcement, network isolation) with detective runtime monitoring, and ships through a GitOps-style CI/CD promotion flow from staging to production.

## Project Objective

The project was built in two phases:

* **Phase 1** - **Deployment.** A multi-tier application running on Kubernetes with Kustomize (base + overlays). The application itself (a multi-tier Java stack) is just a realistic workload — the project is really about the platform around it.
* **Phase 2** - **Security.** Auditing the deployment against DevSecOps practices and working through the gaps one layer at a time: supply chain, pod security, admission control, networking, runtime, and observability.

The goal was to build and secure the infrastructure following DevSecOps best practices across the build, deploy, network, and runtime layers.

### Audit Findings & Remediations

The hardening was driven by a security audit of the Phase 1 deployment, which surfaced findings across several categories. The table summarizes the starting state and the implemented remediation for each.

| Area | Before (Phase 1) | After (Hardened) |
| :--- | :--- | :--- |
| **Image Supply Chain** | Mutable `:latest` tag. Any registry overwrite is silently pulled on the next restart. | **Digest-pinned image** (`@sha256:...`) with explicit pull policy. Cryptographically reproducible; tag overwrite cannot substitute the image. |
| **CI Security** | None — code ships unscanned. | **Blocking Trivy gate** (pinned version). Filesystem and Kubernetes misconfig scans fail the build on CRITICAL/HIGH. |
| **Cluster Access (CI)** | Token-based with TLS verification disabled. | **Full kubeconfig with CA cert** — TLS to the API server verified end-to-end. |
| **Pod Security** | Containers run as root, writable root FS, full capabilities. | **`restricted` profile** on every pod: non-root, read-only root FS, dropped capabilities, seccomp `RuntimeDefault`. |
| **Admission Control** | None — namespace accepts any pod, including privileged. | **Pod Security Admission** (`enforce: restricted`) — non-compliant pods rejected at the API server. |
| **Network** | Flat — every pod can reach every other pod and backend. | **Zero-trust:** `default-deny-all` + explicit allows per flow. Lateral movement denied by default. |
| **Runtime Detection** | None — preventive controls only. | **Falco DaemonSet** watching syscalls, alerting to Telegram on shells, credential reads, and tampering inside app pods. |
| **Observability** | None. | **Prometheus + Grafana + Alertmanager → Telegram**, tuned to suppress k3s false positives and the Watchdog dead-man's-switch. |

## Technology Stack

* **Orchestration:** Kubernetes (k3s)
* **Configuration:** Kustomize (base + staging/prod overlays)
* **Image scanning:** Trivy (CI gate)
* **Admission control:** Pod Security Admission (`restricted`)
* **Network security:** Kubernetes NetworkPolicy (zero-trust)
* **Runtime security:** Falco + falcosidekick
* **Secrets:** Bitnami SealedSecrets
* **Observability:** kube-prometheus-stack (Prometheus, Grafana, Alertmanager)
* **CI/CD:** GitHub Actions
* **Ingress / LB:** ingress-nginx + MetalLB

## Security Posture (Defense in Depth)

The platform layers independent controls, each closing a distinct attack surface:

| Layer | Control | Type |
| :--- | :--- | :--- |
| **Build** | Trivy filesystem + config scan in CI | Preventive |
| **Supply chain** | Image pinned by digest | Preventive |
| **Deploy** | `securityContext` (`restricted`) | Preventive |
| **Admission** | Pod Security Admission | Preventive |
| **Network** | `default-deny` + explicit NetworkPolicies | Preventive |
| **Runtime** | Falco syscall monitoring | **Detective** |

Secrets are managed as code via **SealedSecrets** — encrypted with the controller's public key (safe to commit), decryptable only by the controller inside the cluster. The controller's private key is the single recovery dependency and is kept out-of-band, never in the repository.

## Key Features

* **Digest-pinned images** — the application container is referenced by immutable `@sha256` digest, guaranteeing the exact reviewed image on every deploy and neutralizing tag-overwrite supply-chain attacks.

* **Blocking CI gate** — Trivy runs both filesystem and Kubernetes-misconfig scans, pinned to a release version, failing the pipeline on CRITICAL/HIGH.

* **Hardened pods + admission enforcement** — every workload runs non-root with read-only root FS, dropped capabilities, no privilege escalation, and `RuntimeDefault` seccomp; Pod Security Admission rejects anything non-compliant at the API server (verified with a `Forbidden` on a test pod).

* **Zero-trust networking** — a `default-deny-all` policy blocks all traffic, with explicit allows reconstructing only the required flows (external→web, web→app, app→backends, DNS), so unrelated pods cannot reach each other or unauthenticated backends.

* **Runtime threat detection** — a Falco DaemonSet inspects kernel syscalls and alerts to Telegram on interactive shells, credential-file reads, and tampering inside application pods, scoped to container events to suppress host noise.

* **Environment promotion (CI/CD)** — staging deploys automatically on merge to `main`; the production pipeline triggers only on a published GitHub Release and checks out the exact tagged commit. This separates "what is tested" from "what is promoted", so only a reviewed, tagged state would reach production.

## Known Limitations (intentional, for portfolio scope)

* **TLS on Ingress is not enabled.** Services are served over HTTP on a public `nip.io` host. cert-manager + Let's Encrypt is the planned next step; `nip.io` rate limits on the production ACME issuer make it unreliable for a demo.
* **Grafana admin password.** Ideally this would be a managed secret (a dedicated SealedSecret via `admin.existingSecret`). For this scope it relies on the chart-generated random password — not a default, but not git-managed either.
* **Single notification channel** — staging and production share one Telegram bot/group; a production setup would separate them.
* **Production is modelled, not provisioned.** The prod CD pipeline (`deploy-prod.yaml`) demonstrates the release-driven promotion model — triggered on a GitHub Release, checking out the tagged commit, deploying the `prod` overlay with prod-scoped credentials. The production cluster/namespace itself was not stood up; the focus was building and validating the full DevSecOps posture on staging.
* **Manual image promotion** — production runs the same digest as `base`; promoting a new application version is a manual digest bump rather than an automated trigger from the application CI to this infra repo.
