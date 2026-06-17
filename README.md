# Kubernetes DevSecOps Platform (Kustomize)

![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)
![Kustomize](https://img.shields.io/badge/kustomize-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)
![Falco](https://img.shields.io/badge/falco-%2300AEC7.svg?style=for-the-badge&logo=falco&logoColor=white)
![Trivy](https://img.shields.io/badge/trivy-%231904DA.svg?style=for-the-badge&logo=aqua&logoColor=white)
![Prometheus](https://img.shields.io/badge/prometheus-%23E6522C.svg?style=for-the-badge&logo=prometheus&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/github%20actions-%232671E5.svg?style=for-the-badge&logo=githubactions&logoColor=white)

A multi-tier application deployed on Kubernetes with Kustomize, hardened across the build, deploy, network, and runtime layers. It combines preventive controls (image scanning, pod hardening, admission enforcement, network isolation) with runtime detection via Falco, and promotes from staging to production through GitHub Actions.

## Project Objective

The project was built in two phases.

**Phase 1 — Deployment.** A multi-tier application running on Kubernetes with Kustomize (base + overlays). The application itself (a multi-tier Java stack) is just a realistic workload. The project is really about the platform around it.

**Phase 2 — Security.** Auditing the deployment against DevSecOps practices and working through the gaps one layer at a time: supply chain, pod security, admission control, networking, runtime, and observability.

The goal was to build and secure the infrastructure following DevSecOps best practices across the build, deploy, network, and runtime layers.

### Audit Findings & Remediations

The hardening was driven by an audit of the Phase 1 deployment. The table below pairs each starting-state gap with what was done about it.

| Area | Before (Phase 1) | After (Hardened) |
| :--- | :--- | :--- |
| **Image Supply Chain** | **Mutable `:latest` tag.** Any registry overwrite is silently pulled on the next restart. | **Digest-pinned image.** Pinned by `@sha256` with an explicit pull policy; the exact reviewed image runs every time, and a tag overwrite can't substitute it. |
| **CI Security** | **No scanning.** Code ships unscanned. | **Blocking Trivy gate.** Filesystem and Kubernetes-misconfig scans, pinned to a release version, fail the build on CRITICAL/HIGH. |
| **Cluster Access (CI)** | **TLS verification off.** Token-based access with API-server TLS verification disabled. | **Verified kubeconfig.** Full kubeconfig with the cluster CA, so TLS to the API server is verified. |
| **Pod Security** | **Privileged by default.** Containers run as root with a writable root FS and full capabilities. | **Restricted profile.** Non-root, read-only root FS, dropped capabilities, and seccomp `RuntimeDefault` on every pod. |
| **Admission Control** | **No admission gate.** The namespace accepts any pod, including privileged. | **Pod Security Admission.** `enforce: restricted` rejects non-compliant pods at the API server. |
| **Network** | **Flat network.** Every pod can reach every other pod and backend. | **Zero-trust NetworkPolicies.** `default-deny-all` plus an explicit allow per flow, so unrelated pods can't reach each other. |
| **Runtime Detection** | **No runtime visibility.** Preventive controls only. | **Falco DaemonSet.** Watches syscalls and alerts to Telegram on shells, credential reads, and tampering inside app pods. |
| **Observability** | **No monitoring.** Nothing in place. | **Kube-prometheus-stack.** Prometheus, Grafana, and Alertmanager route to Telegram, tuned to drop k3s false positives and the Watchdog dead-man's-switch. |

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

## Security Posture

The platform stacks independent controls so that no single layer is the only thing standing between an attacker and the workload:

| Layer | Control | Type |
| :--- | :--- | :--- |
| **Build** | Trivy filesystem + config scan in CI | Preventive |
| **Supply chain** | Image pinned by digest | Preventive |
| **Deploy** | `securityContext` (`restricted`) | Preventive |
| **Admission** | Pod Security Admission | Preventive |
| **Network** | `default-deny` + explicit NetworkPolicies | Preventive |
| **Runtime** | Falco syscall monitoring | **Detective** |

Secrets are stored as SealedSecrets, encrypted with the controller's public key so they're safe to commit, and only the controller inside the cluster can decrypt them. The one thing that can't live in git is the controller's private key, which is kept offline as the recovery dependency.

## Key Features

* **Digest-pinned images:** The app container is referenced by `@sha256` digest, not a moving tag, so the same reviewed image runs on every deploy.

* **Blocking CI gate:** Trivy runs a filesystem scan and a Kubernetes-misconfig scan, both pinned to a release version, and fails the pipeline on CRITICAL/HIGH.

* **Hardened pods + admission enforcement:** Every workload runs non-root with a read-only root FS, dropped capabilities, no privilege escalation, and `RuntimeDefault` seccomp. Pod Security Admission then rejects anything non-compliant at the API server. I verified this by trying to run a plain `alpine` pod in the namespace and getting a `Forbidden`.

* **Zero-trust networking:** A `default-deny-all` policy blocks everything, and explicit allows rebuild only the flows that should exist: external to web, web to app, app to its backends, and DNS. Memcached in particular runs without auth, so keeping it unreachable from arbitrary pods matters.

* **Runtime detection:** A Falco DaemonSet reads kernel syscalls and alerts to Telegram on interactive shells, credential-file reads, and log tampering inside app pods. The rules are scoped to container events, otherwise host-level daemons (the DigitalOcean droplet agent, systemd reading `/etc/pam.d`) flood the channel.

* **Release-driven promotion:** Staging deploys on every merge to `main`. Production triggers only on a published GitHub Release and checks out the tagged commit, so a typo merged after the tag never rides into prod. The deploy step also sits behind a GitHub `environment`, which is where a required-reviewer gate would attach.

## Known Limitations

A few things were deliberately left out for the scope of a portfolio project.

* **TLS on Ingress.** Services are served over HTTP on a public `nip.io` host. cert-manager with Let's Encrypt is the obvious next step, but `nip.io` tends to hit rate limits on the production ACME issuer, which makes it unreliable for a demo.

* **Grafana admin password.** This should be a managed secret (a SealedSecret wired in through `admin.existingSecret`). For now it relies on the random password the chart generates. Not a default, but not under git control either.

* **One Telegram channel for both staging and production.** A real setup would split them so prod alerts don't get lost in staging noise.

* **Production is modelled, not provisioned.** `deploy-prod.yaml` shows the promotion model end to end (release trigger, tagged checkout, the `prod` overlay, prod-scoped credentials), but the production cluster was never stood up. The full DevSecOps posture was built and validated on staging.

* **Image promotion is manual.** Production runs the same digest as `base`, so promoting a new version means bumping that digest by hand rather than wiring the application CI to open a PR against this repo.
