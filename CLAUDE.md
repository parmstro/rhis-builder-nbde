# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This Ansible project deploys Network Bound Disk Encryption (NBDE) infrastructure using Tang servers running in Podman containers on RHEL hosts, and configures Clevis clients to bind encrypted devices to those Tang servers.

## Architecture

**Two-tier NBDE deployment:**
- **Tang servers**: Deployed as containers (`rhel8/tang`) managed by systemd on container hosts. The `tang_container` custom role handles container deployment, systemd integration, firewall configuration, SELinux port labeling, and verification testing.
- **Clevis clients**: Configured using the upstream `rhel-system-roles.nbde_client` system role to bind LUKS-encrypted devices to Tang servers.

**Configuration pattern:**
- Container specifications live in `host_vars/<hostname>/containers.yml` with detailed definitions (image, ports, volumes, systemd config, firewall rules, SELinux ports)
- NBDE client bindings live in `host_vars/<hostname>/bindings.yml` specifying devices and Tang server URLs
- Sensitive data (registry credentials, encryption passwords) stored in Ansible vault files referenced via `vault_path` variable

**Deployment flow (tang_container role):**
1. Ensure podman and dependencies installed
2. Pull container images from registry (authenticated)
3. Configure container with systemd integration, firewall, SELinux
4. Verify deployment via clevis encrypt/decrypt round-trip test

## Common Commands

**Deploy Tang servers:**
```bash
ansible-playbook -i inventory \
  -e "vault_path=~/rhisbuilder_vault.yml" \
  --limit containerhosts \
  --vault-password-file=<vault_password_file> \
  containerhost_tang.yml
```

**Configure Clevis clients:**
```bash
ansible-playbook -i inventory \
  -e "vault_path=~/rhisbuilder_vault.yml" \
  --limit clevishosts \
  --vault-password-file=<vault_password_file> \
  clevis_client.yml
```

**Run specific role (development/testing):**
```bash
ansible-playbook -i inventory \
  -e "role_name=tang_container" \
  -e "vault_path=~/rhisbuilder_vault.yml" \
  --limit containerhosts \
  --vault-password-file=<vault_password_file> \
  run_role.yml
```

**Run specific task (development/testing):**
```bash
ansible-playbook -i inventory \
  -e "task_name=ensure_containers" \
  -e "vault_path=~/rhisbuilder_vault.yml" \
  --limit containerhosts \
  --vault-password-file=<vault_password_file> \
  run_task.yml
```

## Key Technical Details

- Tang containers use systemd-managed lifecycle with `generate_systemd` to create `.service` files in `/usr/lib/systemd/system/`
- SELinux port labeling (`tangd_port_t`) required for Tang to bind to non-standard ports
- Firewall rules configured via `ansible.posix.firewalld` module
- Container data persists via named volume `tang-keys:/var/db/tang`
- Deployment verification uses clevis encrypt/decrypt round-trip test against deployed Tang server
- Clevis clients require `rd.neednet=1` kernel parameter in dracut for network-bound decryption during boot

## Inventory Structure

- `containerhosts` group: Hosts running Tang containers
- `clevishosts` group: Hosts with encrypted devices bound to Tang servers

## Dependencies

- Collections: `containers.podman`, `ansible.posix`, `community.general`
- System roles: `rhel-system-roles.nbde_client`
- Runtime dependencies: podman, clevis (for testing), firewalld
