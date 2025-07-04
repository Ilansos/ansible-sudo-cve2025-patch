# Sudo CVE‑2025‑32462 / CVE‑2025‑32463 Auto‑Remediation Playbook

![Ansible Logo](https://raw.githubusercontent.com/ansible/ansible/master/docs/docsite/_static/logo.png)

## 📌 Overview

Two recently disclosed vulnerabilities in **sudo** allow privilege‑escalation attacks on most Linux distributions:

* **CVE‑2025‑32462** – a logic flaw in the *host* option lets authorised users run commands on hosts that were **not** intended for them ([nvd.nist.gov](https://nvd.nist.gov/vuln/detail/CVE-2025-32462?utm_source=chatgpt.com)).  Ubuntu rates this issue as *Low* (CVSS 2.8) ([ubuntu.com](https://ubuntu.com/security/CVE-2025-32462?utm_source=chatgpt.com)).
* **CVE‑2025‑32463** – a path‑traversal weakness when **sudo** is invoked with `--chroot` lets local users gain **root** by loading an attacker‑controlled `nsswitch.conf`; the impact is *Critical* (CVSS 9.3) ([nvd.nist.gov](https://nvd.nist.gov/vuln/detail/CVE-2025-32463?utm_source=chatgpt.com), [ubuntu.com](https://ubuntu.com/security/CVE-2025-32463?utm_source=chatgpt.com)).

Left unpatched, CVE‑2025‑32463 in particular offers adversaries a straightforward route from *any* local account to **root**—a nightmare in large estates where thousands of machines share the same golden image.

This repository ships an **Ansible** playbook that *detects* vulnerable versions and—only after you confirm—*upgrades* sudo to a safe release.

---

## 🛠️ Why Ansible?

[Ansible](https://www.ansible.com/) is an agent‑less automation engine that connects to hosts over SSH (or WinRM) and executes YAML‑described tasks. Because it requires **no pre‑installed client**, you can roll out urgent patches across heterogeneous fleets in minutes instead of days, with an auditable execution log for compliance.

---

## 🚀 Quick Start

### 1 ‑ Clone the repo

```bash
git clone https://github.com/Ilansos/ansible-sudo-cve2025-patch.git && cd ansible-sudo-cve2025-patch.git
```

### 2 ‑ Install requirements

```bash
# If you don't have Ansible installed, run:
# Debian/Ubuntu
sudo apt update
sudo apt install ansible
```

### 3 ‑ Describe your estate in **inventory.yml**

Modify the `inventory.yml` file to include your hosts and credentials. The playbook uses the `lookup` function to read environment variables for sensitive data.

```yaml
servers:
  hosts:
    192.168.1.1:
      ansible_ssh_private_key_file: /path/to/your/private/key
      ansible_user: "{{ lookup('env', 'USER_HOST_1') }}"
      ansible_become_pass: "{{ lookup('env', 'PASSWORD_HOST_1') }}"
    192.168.1.2:
      ansible_ssh_private_key_file: /path/to/your/private/key
      ansible_user: "{{ lookup('env', 'USER_HOST_2') }}"
      ansible_become_pass: "{{ lookup('env', 'PASSWORD_HOST_2') }}"
```

> **Why environment variables?** Credentials never touch disk and can be sourced from a secrets‑manager pipeline.

### 4 ‑ Export the variables

```bash
export USER_HOST_1=admin
export PASSWORD_HOST_1="S3cur3P@ss!"
export USER_HOST_2=admin
export PASSWORD_HOST_2="Diff3rentP@ss!"
```

Repeat for every host entry.

If you manage secrets with **Ansible Vault** or **Doppler**, point the lookup to the vault plugin instead.

### 5 ‑ Run the playbook

```bash
ansible-playbook -i sudo_vuln_patch.yml
```

At the start, the playbook prompts you to confirm whether it should automatically upgrade any hosts it finds with vulnerable sudo versions.

```
One or more hosts may have a vulnerable sudo build.
Do you want to upgrade them automatically? (yes/no)  # default: no
```

Answer `yes` and the patched version is installed via **apt**; answer `no` to perform a harmless audit.

---

## 🔍  What the Playbook Does

| Step | Action                                                                                            | Rationale                                                             |
| ---- | ------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| 1    | `package_facts` gathers the currently installed sudo version                                      | Zero remote commands keep the probe lightweight                       |
| 2    | Builds a distribution key (e.g. `Ubuntu-22.04`) and looks up the **minimum safe version**         | Different distros ship different backports                            |
| 3    | Compares current vs. safe with `dpkg --compare-versions`                                          | Reliable, package‑manager‑native check                                |
| 4    | Reports **⚠ vulnerable** or **✅ already fixed** per host                                         | Immediate visibility                                                  |
| 5    | Updates package lists and upgrades sudo if the user confirmed at the start if he want to upgrade  | Fully unattended remediation                                          |
| 6    | Refreshes facts & re‑checks                                                                       | Proves the upgrade succeeded ✔                                        |

All logic is contained in [`sudo_vuln_patch.yml`](sudo_vuln_patch.yml). Feel free to `--check` for a dry‑run or `--limit webservers` to patch only a subset.

---

## 🏢  The Business Case for Automated Patch Roll‑Outs

* **Scale & speed :** At 100 servers, manual SSH patching at 5 min/host ≈ *8.5 hours of manual labor*. Ansible finishes in under 30 min.
* **Consistency :** The same idempotent tasks run everywhere, eliminating snow‑flake servers.
* **Audit trail :** Play recap plus `--diff` provide evidence for regulators.
* **Reduced dwell time :** Every hour an LPE remains exploitable increases blast radius—automation slashes Mean Time To Remediate (MTTR).

---

## 📄 License

MIT — see `LICENSE` for details.

---

## 🔗 Further Reading

* NVD write‑ups for [CVE‑2025‑32462](https://nvd.nist.gov/vuln/detail/CVE-2025-32462) and [CVE‑2025‑32463](https://nvd.nist.gov/vuln/detail/CVE-2025-32463)
* Ubuntu security notices for [32462](https://ubuntu.com/security/CVE-2025-32462) and [32463](https://ubuntu.com/security/CVE-2025-32463)
* Openwall oss‑security mailing‑list posts (technical root‑cause analysis)

---
