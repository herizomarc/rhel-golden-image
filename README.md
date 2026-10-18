# Automated RHEL 9 Golden Image Factory for Proxmox VE

This repository houses a production-grade **Image-as-Code** pipeline designed to declaratively build, secure, and deploy standardized **Red Hat Enterprise Linux (RHEL) 9** Golden Images directly onto a **Proxmox VE 9** hypervisor cluster.

By leveraging native RHEL compilation utilities and Gitlab-CI, this framework replaces legacy network-heavy Kickstart or ISO orchestration with a streamlined, fast, and optimized disk-appliance deployment model.

## 🛠️ Architecture & Tech Stack

*   **OS Build Engine:** RHEL Image Builder (`osbuild-composer`) — Natively assembles the operating system directly from official Red Hat BaseOS and AppStream RPM repositories.
*   **CI/CD Orchestrator:** GitLab Cloud acting as the control plane, coupled with a hardened, on-premise RHEL 9.8 VM Runner performing local heavy lifting.
*   **Hypervisor Target:** Proxmox VE 9 (Cluster node `pve1`).
*   **Security Compliance:** Operating system volume layouts tailored to meet strict **CIS Benchmarks** and **ANSSI** hardening criteria.
*   **Provisioning Readiness:** Automated pre-installation of `cloud-init` and `qemu-guest-agent` for seamless downstream automated cloning.

## 📁 Repository Structure

```text
├── .gitlab-ci.yml   # Multi-stage hybrid deployment pipeline (2 discrete jobs)
└── blueprint.toml   # Declarative OS blueprint mapping packages, services & CIS partitions
```

## 🔒 Security Hardening & Minimal Privileges

### OS Filesystem Decoupling
To protect target nodes from Denial of Service (DoS) vectors driven by log floods or rogue runtimes, the `blueprint.toml` enforces strict runtime boundary jailing:
*   Dedicated `/var/log` & `/var/log/audit` paths to safeguard forensic trails.
*   Decoupled `/tmp` and `/home` mounts to securely isolate non-root execution frameworks.
*   Zero pre-baked credentials or SSH keys; base authentication handling is entirely deferred to runtime `cloud-init` injections.

### Runner Sudo Cloistering (`/etc/sudoers`)
The GitLab Runner system account (`gitlab-runner`) on the builder machine operates under a strict principle of least privilege, mapped exclusively to the binaries required for the pipeline lifecycle:
```text
gitlab-runner ALL=(ALL) NOPASSWD: /usr/bin/composer-cli, /usr/bin/scp, /usr/bin/rm, /usr/bin/find, /usr/bin/mkdir, /usr/bin/chown
```

## 🚀 Pipeline Lifecycle (2-Stage Architecture)

The workflow is decoupled into two visual jobs within GitLab Cloud while optimizing data transfers over a dedicated high-speed disk partition, circumventing cloud payload upload constraints (GitLab 1GB limits):

### 1. Build Stage (`build_image`)
*   **Prerequisites:** Scans the host system to prune stale historical metadata tracking lists and cleans the runner's dedicated partition space. It also catches and cancels any zombie build stuck in a `RUNNING` state.
*   **Compilation:** Registers the updated `blueprint.toml` structural schema with the host `osbuild` engine, initiates a `.qcow2` build task, and polls the worker engine state in a smart loop until a `FINISHED` signal is raised.
*   **Staging:** Resolves the internal mismatch between the tracking `COMPOSE_UUID` and the internal root-owned `ARTIFACT_UUID` by leveraging the official `image` extraction utility. The raw appliance file is then securely moved onto the heavy dedicated storage partition (`/var/lib/gitlab-runner/share/`).

### 2. Deploy Stage (`deploy_template`)
*   **Acquisition:** Inherits execution telemetry and pulls the heavy appliance straight from the local dedicated runner storage space, bypassing internet bottlenecks.
*   **Local Transfer:** Moves the heavy appliance file onto the Proxmox target node `/tmp/` staging folder using highly secure pre-shared **Ed25519 SSH keys** over the local network interface.
*   **Orchestration:** Connects to the hypervisor to destroy conflicting legacy setups, provision a clean hardware structure with a VirtIO SCSI bus, bind the freshly converted storage partition to `local-lvm`, lock a custom Red Hat Subscription Manager (`rhsm`) bootstrap snippet, and freeze the output into an immutable **VM Template (ID: 9000)**.
*   **Post-Deployment Purge:** Runs a complete cleanup loop inside the final `after_script`, executing a `compose delete` against the worker engine and deep-cleaning system directory remnants to ensure a zero-byte storage overhead footprint between builds.

---
*Maintained as part of an Infrastructure-as-Code (IaC) DevOps automation strategy.*
