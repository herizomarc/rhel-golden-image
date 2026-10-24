# Sovereign RHEL 9 Golden Image Pipeline 📀

An automated DevSecOps pipeline responsible for compiling, hardening, and sealing the base operating system templates utilized across the private enterprise cloud topology.

## 🛠️ Architecture & Tech Stack
* **Image Builder:** Native Red Hat `osbuild-composer` configuration engine.
* **Orchestration:** GitLab-CI pipeline executed on unprivileged Shell Runners.
* **Hypervisor Targeting:** Automated generation and initialization on Proxmox VE.

## 🔐 SecOps Hardening & Sealing
* **Boot-Time Upgrade Prevention:** Implemented `package_upgrade = false` inside the native `[customizations.cloud-init]` schema to completely eliminate dynamic network loops and initialization timeouts at first hardware boot.
* **Automated Package Injection:** Pre-installs critical Ansible prerequisites (`policycoreutils-python-utils`, `firewalld`, `python3-firewall`, `lvm2`).
* **Pristine System Sealing:** Executes programmatic virtualization sanitization routines (`cloud-init clean --logs --seed` and `sys-unconfig`) during template conversion to guarantee immutable cloning nodes.
