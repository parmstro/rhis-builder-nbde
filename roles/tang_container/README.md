# tang_container

Ansible role to deploy Tang server containers using Podman with systemd integration, firewall configuration, and SELinux port labeling.

## Description

This role automates the deployment of Tang servers (Network Bound Disk Encryption key servers) as Podman containers on RHEL container hosts. It handles:

- Installation of Podman and container management tools
- Authenticated container image pulls from registries
- Firewall rule configuration
- SELinux port labeling for Tang daemon
- Systemd service generation for container lifecycle management
- Named volume management for persistent Tang keys
- Deployment verification via clevis encrypt/decrypt testing

## Requirements

- RHEL 8.x or 9.x
- Root access (role uses `become: true`)
- Network access to container registry
- Firewalld service (for firewall configuration)

### Required Collections

- `containers.podman` - Podman container management
- `ansible.posix` - Firewalld module
- `community.general` - SELinux port management

Install with:
```bash
ansible-galaxy collection install containers.podman ansible.posix community.general
```

## Role Variables

### Required Variables

#### `containers`

List of container definitions to deploy. Each container requires the following structure:

```yaml
containers:
  - name: "string"                    # Container name (required)
    image: "string"                   # Container image name without registry (required)
    registry: "string"                # Container registry URL (required)
    registry_username: "string"       # Registry authentication username (required)
    registry_password: "string"       # Registry authentication password (required)
    tag: "string"                     # Image tag (optional, default: "latest")
    id: "string"                      # Container ID (optional)
    force: boolean                    # Force pull image (optional, default: false)
    validate_certs: boolean           # Validate registry SSL certs (optional, default: true)
    detach: boolean                   # Run container detached (optional, default: true)
    state: "string"                   # Container state (optional, default: "started")
    
    # Dependencies
    dependencies: []                  # List of packages to install (optional)
    
    # Firewall configuration
    firewall:                         # List of firewall rules (optional)
      - port: "string"                # Port/service to open (e.g., "8080/tcp")
        zone: "string"                # Firewall zone (e.g., "public")
        state: "string"               # Rule state (e.g., "enabled")
        permanent: boolean            # Make rule permanent (optional, default: true)
    
    # SELinux configuration
    selinux_ports:                    # List of SELinux port labels (optional)
      - ports: integer                # Port number
        protocol: "string"            # Protocol (e.g., "tcp")
        setype: "string"              # SELinux type (e.g., "tangd_port_t")
        state: "string"               # State (e.g., "present")
    
    # Systemd configuration
    generate_systemd:                 # Systemd service generation (optional)
      path: "string"                  # Path to service file (e.g., "/usr/lib/systemd/system/")
      restart_policy: "string"        # Restart policy (e.g., "on-failure")
      time: integer                   # Restart timeout in seconds
      names: boolean                  # Use container name in service file
      container_prefix: "string"      # Prefix for service name (e.g., "rhis")
      wants: "string"                 # Systemd wants dependency (e.g., "network-online.target")
    
    # Container runtime configuration
    publish: []                       # List of port mappings (e.g., ["8080:8080"])
    volume: []                        # List of volume mounts (e.g., ["tang-keys:/var/db/tang"])
    command: "string"                 # Container command override (optional)
    systemd: boolean                  # Enable systemd in container (optional)
```

### Default Variables

See `defaults/main.yml` for default values.

### Internal Variables

The role uses the following internal variables (do not override):
- `test_value` - Value used for Tang server verification testing

## Dependencies

None. The role is self-contained.

## Example Playbook

### Basic Usage

```yaml
---
- name: Deploy Tang Server
  hosts: containerhosts
  become: true
  vars_files:
    - vault.yml  # Contains registry credentials and other secrets
  
  roles:
    - tang_container
```

### Complete Example with Variables

```yaml
---
- name: Deploy Tang Server
  hosts: containerhosts
  become: true
  
  vars:
    containers:
      - name: "tang"
        image: "rhel8/tang"
        id: "tang"
        tag: "latest"
        registry: "registry.redhat.io"
        registry_username: "{{ vault_registry_username }}"
        registry_password: "{{ vault_registry_password }}"
        force: false
        validate_certs: true
        dependencies:
          - clevis
        detach: true
        firewall:
          - port: "8080/tcp"
            zone: "public"
            state: enabled
        generate_systemd:
          path: "/usr/lib/systemd/system/"
          restart_policy: "on-failure"
          time: 120
          names: true
          container_prefix: "rhis"
          wants: "network-online.target"
        publish:
          - "8080:8080"
        selinux_ports:
          - ports: 8080
            protocol: "tcp"
            setype: "tangd_port_t"
            state: present
        state: started
        systemd: true
        volume:
          - "tang-keys:/var/db/tang"
  
  roles:
    - tang_container
```

### Using Host Variables

Define container configurations in `host_vars/<hostname>/containers.yml`:

```yaml
# host_vars/tang1.example.com/containers.yml
containers:
  - name: "tang"
    image: "rhel8/tang"
    registry: "{{ containerhost_registry }}"
    registry_username: "{{ containerhost_registry_username }}"
    registry_password: "{{ containerhost_registry_password }}"
    # ... rest of configuration
```

Then use a simple playbook:

```yaml
---
- name: Deploy Tang Servers
  hosts: containerhosts
  become: true
  
  roles:
    - tang_container
```

## Idempotency

This role is idempotent. Running it multiple times will:
- Skip container pulls if `force: false` and image already exists
- Not recreate containers if configuration hasn't changed
- Not modify firewall rules that already exist
- Not restart services unnecessarily

## Check Mode

**Partial support.** The role supports check mode with the following limitations:
- Firewall and SELinux configuration checks work correctly
- Container operations may report changes even when none would occur
- Verification tasks will be skipped in check mode

## Verification

The role automatically verifies Tang server deployment by:
1. Installing clevis package (if listed in dependencies)
2. Performing encrypt/decrypt round-trip test
3. Asserting decrypted value matches original

If verification fails, the playbook will fail with an assertion error.

## Outputs

The role does not register any variables for use by other roles or tasks.

## Rollback

To remove a Tang container deployment:

1. Stop and remove the container:
```bash
podman stop tang
podman rm tang
```

2. Remove systemd service:
```bash
systemctl stop rhis-tang.service
rm /usr/lib/systemd/system/rhis-tang.service
systemctl daemon-reload
```

3. Remove firewall rules:
```bash
firewall-cmd --permanent --remove-port=8080/tcp --zone=public
firewall-cmd --reload
```

4. Remove SELinux port labeling:
```bash
semanage port -d -t tangd_port_t -p tcp 8080
```

5. Optionally remove named volume (destroys Tang keys):
```bash
podman volume rm tang-keys
```

**Warning:** Removing the `tang-keys` volume will destroy all Tang key material. Encrypted systems bound to this Tang server will not be able to decrypt automatically until rebound.

## Platform Support

| Platform | Version | Status |
|----------|---------|--------|
| RHEL     | 8.x     | ✅ Tested |
| RHEL     | 9.x     | ✅ Tested |
| CentOS Stream | 8 | ⚠️ Should work |
| Fedora   | 38+     | ⚠️ Should work |

## Known Issues

- TODO: Container service startup does not explicitly require firewalld to start first (see configure_container.yml:21)

## Author

parmstro

## License

GPL-3.0

## See Also

- [Tang Documentation](https://github.com/latchset/tang)
- [Clevis Documentation](https://github.com/latchset/clevis)
- [Network Bound Disk Encryption](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/security_hardening/configuring-automated-unlocking-of-encrypted-volumes-using-policy-based-decryption_security-hardening)
