# 🖥️ ChromeCluster — Self-Hosted AI & DevOps Lab on Repurposed Chromeboxes

> A "production-grade" Kubernetes cluster built on 5 refurbished Chromeboxes, running a full self-hosted stack — from AI inference to Git hosting to observability.

---

## What Is This?

Most people spin up a homelab on a Raspberry Pi or rent cloud services. I did it differently: I acquired a handful of end-of-life Chromeboxes, flashed custom firmware to break out of ChromeOS, installed Ubuntu Server, and wired them together into a K3S Kubernetes cluster running real production-grade tooling.

The result is a fully self-hosted platform that runs:
- **Gitea** — private Git server for my projects
- **Tailscale** — zero-config VPN for secure remote access to every service
- **Grafana** — live observability dashboards across all nodes
- **Longhorn** — distributed block storage with replication across nodes
- **Ollama + Open WebUI** — local AI model inference with a browser UI

No monthly cloud bills. No vendor lock-in. Full control over the stack.

---

## Hardware

| Component | Spec |
|-----------|------|
| Nodes | 5× Chromebox (Asus Chromebox 3 |
| Firmware | [MrChromebox UEFI](https://mrchromebox.tech/) — replaces stock firmware |
| OS | Ubuntu Server 26.04 LTS |
| Networking | Gigabit switch, flat LAN + Tailscale overlay |
| Storage | Per-node SSDs, pooled via Longhorn |

The Chromeboxes ship with write-protected firmware designed to only boot ChromeOS. Getting Ubuntu running required:
1. Physically removing the firmware write-protect screw on each unit
2. Flashing MrChromebox's UEFI firmware via recovery mode
3. Netbooting Ubuntu Server and configuring each node for headless operation

This was the most hands-on part of the project — and the most satisfying.

A huge thanks goes to MrChromebox and the amazing work he's done to make flashing custom firmware possible on these devices.

---

## Architecture

```
┌─────────────────────────────────────────────────┐
│                  Tailscale Mesh                 │
│  (encrypted overlay — accessible from anywhere) │
└────────────────────┬────────────────────────────┘
                     │
                     ▼	
        ┌─────────────────────────┐
        │     K3S Control Plane   │
        │       (node-01)         │
        └────────────┬────────────┘
                     │
     ┌───────────────┼───────────────┐
     ▼               ▼               ▼
┌─────────┐    ┌─────────┐    ┌─────────┐
│ node-02 │    │ node-03 │    │ node-04 │    ... node-05
│ Worker  │    │ Worker  │    │ Worker  │
└────┬────┘    └────┬────┘    └────┬────┘
     │              │              │
     └──────────────▼──────────────┘
              Longhorn Storage
          (replicated across nodes)

Services deployed as Kubernetes workloads:
  ├── Gitea          (Git hosting)
  ├── Grafana        (Observability)
  ├── Ollama         (LLM inference)
  └── Open WebUI     (AI chat interface)
```

---

## The Stack

### K3S — Kubernetes Without the Overhead
[K3S](https://k3s.io/) is a lightweight Kubernetes distribution designed for resource-constrained environments. On consumer-grade Chromeboxes without ECC RAM or server-class CPUs, K3S hits the sweet spot: full Kubernetes API compatibility with a fraction of the memory footprint of a standard cluster.

Each node runs as a K3S agent registered to the control plane on node-01. Workloads are scheduled across the worker nodes by the K3S server.

### Longhorn — Distributed Block Storage
The hardest problem in bare-metal Kubernetes is persistent storage. Cloud providers give you a storage class out of the box; on bare metal you build it yourself.

[Longhorn](https://longhorn.io/) creates a distributed storage pool by replicating volumes across multiple nodes. If a node goes offline, the volume stays available from the replicas on other nodes. This gave my cluster proper `ReadWriteOnce` and `ReadWriteMany` PVCs without needing a NAS or external storage appliance.

Lessons learned: Longhorn is sensitive to disk I/O. On slower Chromebox eMMC storage, replica sync during initial volume creation adds noticeable latency. Large and fast SSDs are strongly recommended.

### Tailscale — Zero-Config Secure Networking
[Tailscale](https://tailscale.com/) runs as a daemonset on my seperate networking node and creates a WireGuard-based mesh between all cluster nodes and my personal devices. This means I can reach Gitea, Grafana, and Open WebUI from my laptop anywhere in the world — no port forwarding, no exposed public IPs, no VPN server to manage.

### Gitea — Self-Hosted Git
[Gitea](https://gitea.io/) is a lightweight self-hosted Git service. It backs my personal project repos and doubles as a place to store the manifests and configs that define this cluster.

### Grafana — Observability
[Grafana](https://grafana.com/) with Prometheus scraping node metrics gives me dashboards for CPU, memory, disk I/O, and network across all five nodes. Watching the load shift when Ollama spins up inference on a model was one of the more satisfying moments of this build.

### Ollama + Open WebUI — Local AI Inference
[Ollama](https://ollama.ai/) runs a quantized LLM on the cluster's available CPU resources. Open WebUI provides a ChatGPT-style browser interface that talks to the Ollama API. The model is small enough to run without a GPU — response latency is slower than cloud APIs, but the inference is entirely local, private, and free.

---

## What I Learned

**Bare metal is humbling.** Cloud Kubernetes abstracts away an enormous amount of complexity — networking, storage, node health, certificate management. Doing all of it by hand exposed exactly what those managed services are actually doing for you.

**Distributed storage is hard.** Getting Longhorn stable took the most iteration of anything in this project. Understanding how replication, node failure, and volume attachment interact was a significant learning curve.

**Networking is everything.** K3S needs consistent node-to-node communication. Any flakiness in the LAN — a bad cable, an overloaded switch port — shows up as mysterious pod scheduling failures and etcd timeouts. I learned to diagnose network issues from Kubernetes error messages, which felt like being a forensics detective.

---

## What's Next

- [ ] GitOps with Flux or ArgoCD — have the cluster reconcile itself from the Gitea repo
- [ ] Automated cert rotation with cert-manager + a wildcard cert
- [ ] Add alerting rules to Grafana for node-down conditions
- [ ] Add security protocols

---

## Skills Demonstrated

`Kubernetes` `K3S` `Linux` `Ubuntu Server` `Bare-metal provisioning` `Distributed storage` `Networking` `VPN / WireGuard` `Observability` `Prometheus` `Grafana` `Self-hosted AI` `Git operations` `YAML / Kubernetes manifests` `Hardware troubleshooting` `Firmware flashing`

---
