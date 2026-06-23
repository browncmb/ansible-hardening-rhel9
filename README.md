# Ansible RHEL 9 System Hardening Automation

Ansible automation for applying reusable RHEL 9 hardening controls using firewalld, SELinux, fail2ban SSH protection, Jinja2 templating, handlers, and staged test-to-prod validation.

## Overview

This project builds a Linux system hardening workflow using Ansible roles, firewalld, SELinux, and fail2ban. The automation applies firewall service rules, enforces SELinux settings, configures selected SELinux booleans, deploys fail2ban SSH protection, and validates that hardened systems remain accessible through Ansible and FreeIPA-based authentication workflows.

The goal is to demonstrate repeatable Linux hardening automation with a focus on RHEL infrastructure operations, role-based Ansible design, idempotent configuration, staged validation, and production-minded change control.

This project builds on the FreeIPA/NFS lab environment documented in `ansible-freeipa-nfs` and complements the baseline STIG scanning workflow documented in `ansible-stig-rhel9`. It does not claim full STIG remediation. Instead, it automates selected practical hardening controls and validates their operational impact in a FreeIPA-based Linux environment.

## Architecture

```text
                         +-----------------------------+
                         |       Ansible Control       |
                         |   ansible01.example.test    |
                         +--------------+--------------+
                                        |
                                        | Ansible SSH
                                        |
              +-------------------------+--------------------------+
              |                         |                          |
              v                         v                          v
+-----------------------------+ +-----------------------------+ +-----------------------------+
| Test Hardening Target       | | Prod Hardening Target       | | Prod Hardening Target       |
| client01.example.test       | | client02.example.test       | | client03.example.test       |
|                             | |                             | |                             |
| - firewalld service rules   | | - firewalld service rules   | | - firewalld service rules   |
| - SELinux enforcement       | | - SELinux enforcement       | | - SELinux enforcement       |
| - fail2ban SSH protection   | | - fail2ban SSH protection   | | - fail2ban SSH protection   |
| - FreeIPA client context    | | - FreeIPA client context    | | - FreeIPA client context    |
| - SSH/sudo access           | | - SSH/sudo access           | | - SSH/sudo access           |
+-----------------------------+ +-----------------------------+ +-----------------------------+
```

## Technologies Used

* Red Hat Enterprise Linux 9
* Ansible
* Ansible roles
* firewalld
* SELinux
* fail2ban
* EPEL
* Jinja2 templates
* SSH
* sudo
* FreeIPA client context
* NFS/autofs validation context
* Git/GitHub

## Repository Structure

```text
.
├── ansible.cfg
├── docs/
│   └── idempotency_test.md
├── inventory/
│   └── hosts.ini
├── playbooks/
│   ├── harden.yml
│   └── site.yml
├── roles/
│   └── hardening/
│       ├── defaults/
│       │   └── main.yml
│       ├── handlers/
│       │   └── main.yml
│       ├── tasks/
│       │   └── main.yml
│       ├── templates/
│       │   └── hardening.conf.j2
│       └── vars/
│           └── main.yml
├── requirements.yml
├── LICENSE
└── README.md
```

## Inventory Design

The inventory separates managed systems into `test` and `prod` groups. The `hardening_targets` group includes both groups so the same hardening workflow can target all systems or be limited to a specific environment group.

```ini
[test]
client01.example.test

[prod]
client02.example.test
client03.example.test

[hardening_targets:children]
test
prod
```

This structure supports staged rollout. The role can be tested against a single test host before being promoted to prod hosts.

## What This Automation Does

The automation performs the following:

* Installs base hardening packages
* Ensures firewalld is running and enabled
* Allows required firewall services
* Removes blocked firewall services
* Enforces SELinux state and policy
* Configures selected SELinux booleans
* Configures EPEL package access for fail2ban installation
* Installs fail2ban and firewalld integration packages
* Deploys a Jinja2 fail2ban jail configuration for SSH protection
* Configures fail2ban to monitor SSH authentication events through journald
* Starts and enables fail2ban
* Uses handlers to reload or restart services only when related configuration changes
* Validates test-to-prod hardening behavior
* Confirms idempotency on repeated runs

## Ansible Design

This project uses a focused playbook and a master workflow playbook to keep orchestration separate from role implementation.

| Playbook     | Purpose                                            |
| ------------ | -------------------------------------------------- |
| `harden.yml` | Applies the hardening role to managed systems      |
| `site.yml`   | Provides the master hardening workflow entry point |

The `hardening` role manages the system hardening implementation.

Key role functions include:

* Managing firewalld service rules
* Enforcing SELinux mode and policy
* Managing SELinux booleans
* Installing EPEL-backed fail2ban packages
* Rendering the fail2ban SSH jail configuration from a Jinja2 template
* Restarting fail2ban only when the jail configuration changes
* Reloading firewalld only when firewall rule changes are made
* Supporting targeted execution through role-level tags

Role-level tags include:

| Tag        | Purpose                                 |
| ---------- | --------------------------------------- |
| `firewall` | Runs firewalld-related hardening tasks  |
| `selinux`  | Runs SELinux-related hardening tasks    |
| `fail2ban` | Runs fail2ban SSH protection tasks      |
| `packages` | Runs package and dependency setup tasks |

## Role Structure

| Directory    | Purpose                                                                                                |
| ------------ | ------------------------------------------------------------------------------------------------------ |
| `defaults/`  | Stores configurable role defaults such as firewall services, SELinux settings, and fail2ban thresholds |
| `vars/`      | Stores internal role values such as package lists, EPEL repository details, and destination paths      |
| `tasks/`     | Contains the firewalld, SELinux, and fail2ban hardening tasks                                          |
| `handlers/`  | Contains service reload and restart handlers                                                           |
| `templates/` | Contains the fail2ban jail configuration template                                                      |

## Role Defaults

The following defaults are defined in `roles/hardening/defaults/main.yml`.

| Variable                              | Default                                                                               | Description                                               |
| ------------------------------------- | ------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| `hardening_firewall_zone`             | `public`                                                                              | firewalld zone managed by the role                        |
| `hardening_firewall_services_allowed` | `ssh`, `freeipa-ldap`, `freeipa-ldaps`, `dns`, `nfs`                                  | Firewall services that should be enabled                  |
| `hardening_firewall_services_blocked` | `telnet`, `ftp`                                                                       | Firewall services that should be disabled                 |
| `hardening_selinux_state`             | `enforcing`                                                                           | Desired SELinux state                                     |
| `hardening_selinux_policy`            | `targeted`                                                                            | Desired SELinux policy                                    |
| `hardening_selinux_booleans`          | `use_nfs_home_dirs=true`, `ssh_sysadm_login=false`, `httpd_can_network_connect=false` | SELinux booleans managed by the role                      |
| `hardening_fail2ban_bantime`          | `3600`                                                                                | Number of seconds an IP remains banned                    |
| `hardening_fail2ban_findtime`         | `600`                                                                                 | Time window used to count failed SSH attempts             |
| `hardening_fail2ban_maxretry`         | `5`                                                                                   | Number of failed attempts before a ban is applied         |
| `hardening_fail2ban_ignoreip`         | `[]`                                                                                  | Optional trusted IP addresses excluded from fail2ban bans |

## fail2ban SSH Protection

The role deploys a fail2ban jail configuration for SSH protection using a Jinja2 template.

The template configures fail2ban to use the systemd backend and monitor SSH authentication events through the journal on RHEL 9 systems. The `ignoreip` setting includes loopback addresses, the managed host’s primary IPv4 address, and optional trusted IP overrides.

Manual validation confirmed that fail2ban can delegate SSH bans to firewalld through rich rules and remove those rules after unbanning the test IP.

## Ansible Collection Dependencies

Required Ansible collections are defined in `requirements.yml`.

Install the collections locally with:

```bash
ansible-galaxy collection install -r requirements.yml -p collections/
```

The `collections/` directory is used as a local dependency install path and is not intended to be committed to the repository.

## Prerequisites

To run this project, the environment should have:

* RHEL 9 systems with active subscriptions
* SSH access from the Ansible control node to managed nodes
* Passwordless sudo or appropriate privilege escalation configured
* Python available on managed nodes
* Ansible installed on the control node
* Required Ansible collections installed from `requirements.yml`
* Inventory configured with test and prod groups
* EPEL access available for fail2ban package installation

## Running the Automation

Run commands from the repository root.

### Syntax check

```bash
ansible-playbook playbooks/site.yml --syntax-check
```

### Run the full hardening workflow

```bash
ansible-playbook playbooks/site.yml
```

### Run against the test group

```bash
ansible-playbook playbooks/site.yml -l test
```

### Run against the prod group

```bash
ansible-playbook playbooks/site.yml -l prod
```

### Run only firewalld hardening against the test group

```bash
ansible-playbook playbooks/site.yml --tags firewall -l test
```

### Run only SELinux hardening against the test group

```bash
ansible-playbook playbooks/site.yml --tags selinux -l test
```

### Run only fail2ban hardening against the test group

```bash
ansible-playbook playbooks/site.yml --tags fail2ban -l test
```

### Run package and dependency setup against the test group

```bash
ansible-playbook playbooks/site.yml --tags packages -l test
```

## Validation

The hardening workflow validated that:

* Required firewalld services were present after hardening
* Blocked services were not listed in firewalld
* SELinux was enforcing on managed systems
* Selected SELinux booleans were applied
* The fail2ban service was installed, running, and enabled
* The `sshd` fail2ban jail was active
* SSH authentication events were monitored through journald
* Manual SSH bans were delegated to firewalld through rich rules
* Test bans were removed after unbanning
* Ansible control access remained available after hardening
* FreeIPA user authentication still worked after hardening
* FreeIPA identity resolution and NFS/autofs home directory access remained functional

## Idempotency

Idempotency testing confirmed that the hardening role can be reapplied safely after the target system already matches the desired configuration.

The second run completed with `changed=0`, confirming that the role does not repeatedly modify an already configured system.

Detailed idempotency results are documented in [idempotency_test.md](docs/idempotency_test.md).

## Skills Demonstrated

This project demonstrates:

* RHEL 9 system hardening automation
* Ansible role development
* firewalld service management
* SELinux enforcement and boolean management
* fail2ban SSH protection
* EPEL-based package installation
* Jinja2 template deployment
* Handler-based service management
* Ansible tag-based execution
* Inventory-based test-to-prod promotion
* Idempotency testing
* Operational validation after hardening
* FreeIPA authentication impact validation
* Git-based infrastructure documentation

## Result

This project provides a repeatable Ansible workflow for applying selected Linux system hardening controls across RHEL 9 lab systems. The final workflow ties together firewalld service control, SELinux enforcement, fail2ban SSH protection, idempotent role design, and staged test-to-prod validation while confirming that Ansible control access, FreeIPA authentication, and NFS/autofs home directory access remain functional after hardening.
