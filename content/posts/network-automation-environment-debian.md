---
title: "Building a Network Automation Environment on Debian: Python, Ansible, Terraform, and Git"
date: 2026-05-31
draft: true
tags:
  - network-automation
  - ansible
  - terraform
  - python
  - git
  - debian
  - molecule
  - gitops
  - nornir
categories: ["explainer", "automation"]
summary: "A from-scratch build of a network automation environment on Debian — Python, Ansible, Terraform, Git — plus the testing, collaboration, and monitoring layers, with the common how-to-guide traps called out where they bite."
description: "Build a network automation environment on Debian — Python, Ansible, Terraform, Git, plus Molecule testing, collaboration, and monitoring, common traps fixed."
---

This is the build I'd hand a new engineer setting up a network automation environment from a clean Debian box: Python, Ansible, Terraform, and Git, then the layers most guides skip — testing, collaboration, and monitoring. It's a how-to, not a war story, so treat it as a reference you can copy from.

A warning up front, because it's the thing that wastes the most time: several popular automation tutorials repeat steps that no longer work — a deprecated Ansible VLAN module, a Cisco "Docker image" that doesn't exist, a broken Molecule install combination. I've flagged each one inline as a **Trap**, with what actually works. If you've followed an older guide and something silently failed, one of these is probably why.

## What you need

- A fresh Debian 11 or 12 server, sudo access.
- Working knowledge of networking and the Linux command line.

## Phase 1 — Environment setup

### System prep

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git python3-pip python3-venv sshpass
```

### Python and a virtual environment

Always work inside a venv — it keeps Ansible, Molecule, and their dependencies off the system Python, which matters the moment you have two projects on different collection versions.

```bash
python3 --version
python3 -m venv ~/venv/network-automation
source ~/venv/network-automation/bin/activate
```

### Ansible

Install Ansible into the venv with pip rather than apt — the apt package lags, and network collections move fast.

```bash
pip install ansible
ansible --version
ansible-galaxy collection install cisco.ios arista.eos junipernetworks.junos
```

### Terraform

```bash
sudo apt install -y gnupg software-properties-common
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y terraform
terraform --version
```

### Git

```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
git config --global core.editor "nano"
```

## Phase 2 — Directory structure

A predictable layout pays off the day someone else has to find something.

```bash
mkdir -p ~/network-automation/{ansible,inventory,terraform,python-scripts,git-repos,documentation}
cd ~/network-automation
```

```text
network-automation/
├── ansible/         (playbooks/ roles/ group_vars/ host_vars/)
├── inventory/       (production/ staging/ lab/)
├── terraform/       (modules/ aws/ gcp/ azure/)
├── python-scripts/  (network-tools/ api-integrations/ utilities/)
├── git-repos/       (network-configs/ automation-scripts/)
└── documentation/   (network-diagrams/ procedures/)
```

## Phase 3 — Initial configuration

### Ansible config

```bash
mkdir -p ~/network-automation/ansible
cat > ~/network-automation/ansible/ansible.cfg <<'EOF'
[defaults]
inventory = ../inventory/
host_key_checking = False
gathering = explicit
retry_files_enabled = False
log_path = ~/network-automation/logs/ansible.log

[persistent_connection]
connect_timeout = 60
command_timeout = 60
EOF
```

`host_key_checking = False` is convenient in a lab and a liability in production — re-enable it once your known_hosts is seeded. I've added `log_path` here from the start; the monitoring phase later depends on Ansible actually writing a log.

### Inventory and group vars

```bash
mkdir -p ~/network-automation/inventory/production
cat > ~/network-automation/inventory/production/hosts <<'EOF'
[cisco_routers]
router1 ansible_host=192.0.2.1
router2 ansible_host=192.0.2.2

[cisco_switches]
switch1 ansible_host=192.0.2.101
switch2 ansible_host=192.0.2.102

[network:children]
cisco_routers
cisco_switches
EOF
```

```bash
mkdir -p ~/network-automation/ansible/group_vars
cat > ~/network-automation/ansible/group_vars/cisco_routers <<'EOF'
---
ansible_network_os: cisco.ios.ios
ansible_user: admin
ansible_password: "{{ vault_ansible_password }}"
ansible_connection: ansible.netcommon.network_cli
EOF
```

> **Trap — the connection plugin name.** Use `ansible.netcommon.network_cli`, the fully-qualified name. The bare `network_cli` still resolves today but is the old short form; spell it out so it doesn't break when short-name resolution changes.

## Phase 4 — Basic automation examples

### Python: a device backup script

A small Paramiko script to pull a running-config. It's deliberately basic — for real fleet backup I'd reach for Netmiko or Nornir, which handle enable mode and paging properly. Two things this version gets right that the common copy-paste version gets wrong: it imports `time` (the usual version calls `time.sleep` without importing it and crashes), and it reminds you that interactive `invoke_shell` is fragile.

```python
#!/usr/bin/env python3
import paramiko
import time
from datetime import datetime
import getpass


def backup_device(host, username, password, enable_password):
    ssh = paramiko.SSHClient()
    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    try:
        ssh.connect(host, username=username, password=password, look_for_keys=False)
        chan = ssh.invoke_shell()
        chan.send('enable\n')
        chan.send(f'{enable_password}\n')
        chan.send('terminal length 0\n')
        chan.send('show running-config\n')
        time.sleep(2)
        output = chan.recv(65535).decode('utf-8')
        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        filename = f"{host}_config_{timestamp}.txt"
        with open(filename, 'w') as f:
            f.write(output)
        print(f"Backup saved to {filename}")
    except Exception as e:
        print(f"Error connecting to {host}: {str(e)}")
    finally:
        ssh.close()


if __name__ == "__main__":
    host = input("Device IP: ")
    username = input("Username: ")
    password = getpass.getpass("Password: ")
    enable_password = getpass.getpass("Enable Password: ")
    backup_device(host, username, password, enable_password)
```

```bash
pip install paramiko
```

`set_missing_host_key_policy(AutoAddPolicy())` blindly trusts unknown host keys — fine for a lab, not for production. The `time.sleep(2)` is also a guess; on a slow or large device the config gets truncated. This is exactly why the Ansible/Netmiko path is better for anything real.

### Ansible: gather facts

```bash
mkdir -p ~/network-automation/ansible/playbooks
cat > ~/network-automation/ansible/playbooks/gather_facts.yml <<'EOF'
---
- name: Gather network device facts
  hosts: network
  gather_facts: false
  tasks:
    - name: Gather device facts
      cisco.ios.ios_facts:
      register: facts
    - name: Display facts
      ansible.builtin.debug:
        var: facts
EOF
```

```bash
ansible-playbook ~/network-automation/ansible/playbooks/gather_facts.yml \
  -i ~/network-automation/inventory/production/hosts
```

### Terraform: an AWS VPC

```bash
mkdir -p ~/network-automation/terraform/aws/vpc
cd ~/network-automation/terraform/aws/vpc
cat > main.tf <<'EOF'
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "NetworkAutomationVPC" }
}

resource "aws_subnet" "main" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
  tags = { Name = "MainSubnet" }
}
EOF
terraform init
terraform plan
# terraform apply   # when you actually mean it
```

I bumped the AWS provider to `~> 5.0` — the `4.0` pin many guides still show is two majors behind and will pull a years-old provider.

## Phase 5 — Git

```bash
cd ~/network-automation
git init
cat > .gitignore <<'EOF'
# Ansible vault
*.vault
# Terraform state — never commit this
*.tfstate
*.tfstate.*
.terraform/
# Python venv
venv/
# Secrets
*.key
*.pem
*.secret
EOF
git add .
git commit -m "Initial commit of network automation framework"
```

Committing Terraform state is a classic and dangerous mistake — `*.tfstate` can contain secrets in plaintext. The `.gitignore` above blocks it; keep it that way and use remote state for anything shared.

## Phase 6 — Secrets, hooks, scheduled backups

### Ansible Vault

```bash
ansible-vault create ~/network-automation/ansible/group_vars/vault
```

```yaml
---
vault_ansible_password: "your_secure_password"
```

### A pre-commit hook that actually validates

```bash
cat > ~/network-automation/.git/hooks/pre-commit <<'EOF'
#!/bin/bash
set -e

echo "Validating Ansible syntax..."
find ansible/playbooks -name "*.yml" -type f | while read -r file; do
    ansible-playbook --syntax-check "$file"
done

echo "Validating Terraform formatting..."
terraform -chdir=terraform fmt -check -recursive

echo "Validating Python syntax..."
find python-scripts -name "*.py" -type f | while read -r file; do
    python3 -m py_compile "$file"
done
EOF
chmod +x ~/network-automation/.git/hooks/pre-commit
```

> **Trap — `terraform fmt -check "$file"`.** `terraform fmt` operates on a directory, not a single file path piped in one at a time. Use `fmt -check -recursive` against the terraform directory (shown above), or it'll error or silently check nothing.

### Scheduled backups with cron

```bash
(crontab -l 2>/dev/null; echo "0 2 * * * /usr/bin/ansible-playbook /home/$USER/network-automation/ansible/playbooks/backup_configs.yml") | crontab -
```

Cron won't have your venv activated — point the cron line at the venv's `ansible-playbook` (`/home/$USER/venv/network-automation/bin/ansible-playbook`), not `/usr/bin`, or it runs the wrong interpreter.

## Phase 7 — Testing, collaboration, monitoring

This is where most guides go wrong, so it's worth getting right.

### Testing with Molecule — what it can and can't do

Molecule is excellent for testing the *logic* of an Ansible role — does it template correctly, is it idempotent, does it handle variables — using throwaway Docker containers. What it **cannot** do is emulate a Cisco router in a Docker container.

> **Trap — the Cisco IOS-XE "Docker image."** A lot of tutorials tell you to set `image: cisco/cisco-ios-xe:16.12.03` and test your network role against it with Molecule's docker driver. That image does not exist on Docker Hub, and even if it did, Molecule's docker driver spins up Linux containers — it can't run IOS's CLI or SSH stack. The whole "Molecule against a Cisco container" workflow doesn't run. IOS-XE is a full network OS; you virtualize it as a VM, not a container.

So split testing into the two things you actually want:

**1. Test role logic with Molecule (real, works today).** Install it correctly first:

> **Trap — the Molecule install.** Don't run `pip install molecule molecule-docker molecule-ansible`. `molecule-docker` is the deprecated standalone package, `molecule-ansible` isn't a real package, and mixing the old standalone with the new plugins gives you a broken setup (duplicate entry points). The current, correct install is:

```bash
pip install "molecule-plugins[docker]"
molecule --version
```

Scaffold and run a role test on a Linux container — this validates templating and idempotence, which is most of what breaks:

```bash
cd ~/network-automation/ansible/roles
ansible-galaxy role init ios_config
cd ios_config
molecule init scenario -d docker
molecule test   # create → converge → idempotence → verify → destroy
```

**2. Test against real IOS-XE with a virtual device.** When you need to confirm config actually applies to IOS, run a *virtual* IOS-XE node — Catalyst 8000v (`c8000v`) or CSR1000v — built as a VM image and launched through [vrnetlab](https://github.com/hellt/vrnetlab) inside [containerlab](https://containerlab.dev/manual/kinds/vr-c8000v/), or in [Cisco Modeling Labs (CML)](https://developer.cisco.com/docs/modeling-labs/). Then point Ansible (or Nornir) at the lab node and run the same role. That's the workflow that actually validates device behavior — it's just heavier than a container, because a real network OS is heavier than a container.

> **Trap — `cisco.ios.ios_vlan`.** The singular `ios_vlan` (and `ios_vlans`-vs-`ios_vlan` confusion) trips people up: `cisco.ios.ios_vlan` was **deprecated and removed after 2022-06-01**. Use the resource module `cisco.ios.ios_vlans` (plural), which takes a `config:` list, not the old flat `vlan_id`/`name` arguments. A current `cisco.ios` collection will fail outright on `ios_vlan`.

A correct VLAN role task today looks like:

```yaml
- name: Configure VLANs
  cisco.ios.ios_vlans:
    config:
      - name: TEST_VLAN
        vlan_id: 10
    state: merged
```

### Collaboration — Gitea or GitHub/GitLab

For a self-hosted option, Gitea runs comfortably on the same Debian box. Pin a current release rather than copy-pasting an old version number:

```bash
sudo apt install -y sqlite3
# Check https://dl.gitea.com for the latest 1.2x release and substitute below
VERSION=1.21.11
wget -O gitea "https://dl.gitea.com/gitea/${VERSION}/gitea-${VERSION}-linux-amd64"
chmod +x gitea && sudo mv gitea /usr/local/bin/
sudo adduser --system --group --disabled-password --home /var/lib/gitea git
sudo mkdir -p /var/lib/gitea/{custom,data,log} /etc/gitea
sudo chown -R git:git /var/lib/gitea/
sudo chown -R git:git /etc/gitea
```

```bash
sudo tee /etc/systemd/system/gitea.service > /dev/null <<'EOF'
[Unit]
Description=Gitea
After=network.target

[Service]
User=git
Group=git
WorkingDirectory=/var/lib/gitea
ExecStart=/usr/local/bin/gitea web --config /etc/gitea/app.ini
Restart=always
Environment=GITEA_WORK_DIR=/var/lib/gitea

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable --now gitea
```

Finish setup at `http://<server-ip>:3000`. The `dl.gitea.io` host in older guides now redirects to `dl.gitea.com`, and `sudo cat >` doesn't elevate the redirect — use `sudo tee` (as above) or you'll get a permission-denied writing the unit file.

A branching model that holds up: `main` protected and production-ready, `dev` for integration, `feature/*` and `fix/*` for work. Require review on `main`, and gate merges on the Molecule role tests passing in CI (GitHub Actions or GitLab CI runs the same `molecule test` you run locally).

### Monitoring automation jobs

The thing to understand: **Ansible is a batch job, not a long-running service, so Prometheus can't scrape it directly.** 

> **Trap — scraping Ansible.** Configs that point Prometheus at `localhost:9090` with a `/ansible_metrics` path are scraping Prometheus itself and hitting a path nothing serves. There's no Ansible HTTP endpoint to scrape.

Two honest approaches:

**Push metrics from your runs.** Wrap playbook runs in a small script that records duration and pass/fail and pushes them to a Prometheus Pushgateway (Pushgateway exists precisely for batch jobs). Then Prometheus scrapes the Pushgateway, and Grafana charts the trend. The metrics you'd track — failure count, run duration, changed-task count — only exist because you emit them; no off-the-shelf dashboard ID conjures them. (If you import a community dashboard, confirm what metrics it expects rather than trusting an ID from a blog — several widely-copied IDs don't match what people claim.)

**Or just centralize the logs (simpler, and enough for most teams).** You already set `log_path` in `ansible.cfg` back in Phase 3. Ship that log with Promtail into Loki, and alert on `ERROR` lines — or use the ELK stack if that's what you run. For a small team this catches the failures that matter without standing up a metrics pipeline.

## Where to take it next

Expand the inventory to more device types; build role-level tests for your common changes; once you have a few real automation incidents behind you, write *those* up — the snapshot-before-changes discipline, the config push that didn't take, the drift you caught. The framework is the easy part. The judgment about when and how to use it is what's worth sharing.

---

*Technical references: [Ansible cisco.ios.ios_vlans module](https://docs.ansible.com/projects/ansible/latest/collections/cisco/ios/ios_vlans_module.html); [molecule-plugins (Docker driver)](https://github.com/ansible-community/molecule-plugins); [containerlab — Cisco c8000v / vrnetlab](https://containerlab.dev/manual/kinds/vr-c8000v/); [Cisco Modeling Labs](https://developer.cisco.com/docs/modeling-labs/); [Terraform AWS provider](https://registry.terraform.io/providers/hashicorp/aws/latest); [Gitea downloads](https://dl.gitea.com).*
