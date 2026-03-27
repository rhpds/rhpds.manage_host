# OPA Setup Role (Podman Quadlet)

This Ansible role installs and configures Open Policy Agent (OPA) as a containerized service using Podman Quadlet.

## Description

The `opa_setup` role performs the following tasks:

1. **Podman Installation** - Ensures Podman is installed
2. **Directory Structure** - Creates required directories for config, data, policies, and logs
3. **Container Image** - Pulls the official OPA container image
4. **Quadlet Configuration** - Generates Podman Quadlet `.container` file
5. **Systemd Service** - Systemd automatically manages the container as a service via Quadlet
6. **Health Checking** - Verifies OPA service is running and healthy

## What is Podman Quadlet?

Quadlet is a systemd generator that creates systemd services from Podman container definitions. It provides:

- **Declarative container management** using `.container` files
- **Native systemd integration** with automatic service creation
- **Rootless or rootful** container execution
- **Auto-updates** and health checks
- **Dependency management** between containers

## Requirements

- Ansible 2.9 or higher
- Target system: RHEL/CentOS 8/9 or compatible
- Podman 4.4+ (with Quadlet support)
- Systemd-based system
- Internet access to pull container images

## Role Variables

### Container Image Variables

```yaml
# OPA container image
opa_container_image: "docker.io/openpolicyagent/opa:0.60.0"
opa_pull_policy: "missing"
```

### Directory Variables

```yaml
# Directory paths
opa_config_dir: "/etc/opa"
opa_data_dir: "/var/lib/opa"
opa_log_dir: "/var/log/opa"
opa_policy_dir: "/var/lib/opa/policies"
```

**Note**: No system user/group is created. The OPA container runs with its default non-root user (UID 1000), and directories are created with world-writable permissions for the container to access.

### Quadlet Configuration

```yaml
# Quadlet directory and file
opa_quadlet_dir: "/etc/containers/systemd"
opa_quadlet_file: "{{ opa_quadlet_dir }}/opa.container"

# Service and container names
opa_service_name: opa
opa_container_name: opa
```

### Service Configuration

```yaml
# Service state
opa_service_enabled: true
opa_service_state: started

# Network ports
opa_server_port: 8181
opa_diagnostic_port: 8282

# Published ports
opa_publish_ports:
  - "{{ opa_server_port }}:8181"
  - "{{ opa_diagnostic_port }}:8282"
```

### Container Configuration

```yaml
# Restart policy: always, on-failure, no
opa_restart_policy: "always"

# Timezone
opa_timezone: "UTC"

# SELinux labels for volumes
opa_volume_selinux_label: "Z"

# Optional: Specify container user (empty = use container default)
opa_container_user: ""
```

### Bundle Configuration (Optional)

```yaml
# Enable bundle management
opa_bundle_enabled: false
opa_bundle_service: ""
opa_bundle_resource: ""
opa_bundle_polling_min_delay: 60
opa_bundle_polling_max_delay: 120
```

### Decision Logs (Optional)

```yaml
# Enable decision logging
opa_decision_logs_enabled: false
opa_decision_logs_service: ""
opa_decision_logs_resource: ""
```

### Custom Arguments

```yaml
# Additional OPA command-line arguments
opa_extra_args:
  - "--log-level=debug"
  - "--log-format=json"
```

## Dependencies

None.

## Example Playbook

### Basic Installation

```yaml
---
- name: Install OPA with Quadlet
  hosts: opa_servers
  become: true

  roles:
    - opa_setup
```

### Custom Configuration

```yaml
---
- name: Install OPA with custom configuration
  hosts: opa_servers
  become: true

  vars:
    opa_container_image: "docker.io/openpolicyagent/opa:0.60.0"
    opa_server_port: 8181
    opa_diagnostic_port: 8282

    # Enable bundle management
    opa_bundle_enabled: true
    opa_bundle_service: "https://bundle-server.example.com"
    opa_bundle_resource: "/bundles/authz.tar.gz"

    # Enable decision logs
    opa_decision_logs_enabled: true
    opa_decision_logs_service: "https://decision-logs.example.com"

    # Custom logging
    opa_extra_args:
      - "--log-level=info"
      - "--log-format=json"

  roles:
    - opa_setup
```

## Post-Installation

### Verify Installation

```bash
# Check Quadlet file
cat /etc/containers/systemd/opa.container

# Check service status
systemctl status opa

# View container
podman ps --filter name=opa

# View logs
journalctl -u opa -f
# or
podman logs -f opa

# Health check
curl http://localhost:8181/health
```

### Manage Policies

```bash
# Add a policy via API
curl -X PUT http://localhost:8181/v1/policies/example \
  --data-binary @policy.rego

# List policies
curl http://localhost:8181/v1/policies

# Query a policy
curl -X POST http://localhost:8181/v1/data/example/allow \
  -H 'Content-Type: application/json' \
  -d '{"input": {"user": "alice"}}'
```

### Service Management

```bash
# Start service (starts container)
systemctl start opa

# Stop service (stops container)
systemctl stop opa

# Restart service
systemctl restart opa

# Enable on boot
systemctl enable opa

# Reload Quadlet configuration
systemctl daemon-reload
```

### Container Management

```bash
# View container status
podman ps --filter name=opa

# View container logs
podman logs opa
podman logs -f opa  # Follow logs

# Execute command in container
podman exec opa opa version

# Inspect container
podman inspect opa

# Update container image
podman pull docker.io/openpolicyagent/opa:latest
systemctl restart opa
```

## Directory Structure

After installation, the following structure is created:

```
/etc/opa/                     # Configuration files
  └── config.yaml             # Main OPA configuration
/var/lib/opa/                 # Data directory (mounted to container)
  └── policies/               # Policy files
/var/log/opa/                 # Log directory (mounted to container)
/etc/containers/systemd/      # Quadlet directory
  └── opa.container           # Quadlet container definition
```

**Note**: Directories are created with 0777 permissions to allow the container's default user (UID 1000) to write to them. In production, consider more restrictive permissions or using user namespaces.

## API Endpoints

Once OPA is running, the following endpoints are available:

- **Health**: `http://localhost:8181/health`
- **Policies**: `http://localhost:8181/v1/policies`
- **Data**: `http://localhost:8181/v1/data`
- **Query**: `http://localhost:8181/v1/query`
- **Diagnostics**: `http://localhost:8282/metrics`

## Quadlet File Format

The generated Quadlet file (`/etc/containers/systemd/opa.container`) follows this format:

```ini
[Unit]
Description=Open Policy Agent (OPA) Container
After=network-online.target

[Container]
Image=docker.io/openpolicyagent/opa:0.60.0
ContainerName=opa
Volume=/etc/opa/config.yaml:/config/config.yaml:ro,Z
Volume=/var/lib/opa:/data:Z
PublishPort=8181:8181
PublishPort=8282:8282
Exec=run --server --config-file=/config/config.yaml
AutoUpdate=registry
HealthCmd=curl -f http://localhost:8181/health || exit 1

[Service]
Restart=always

[Install]
WantedBy=multi-user.target
```

## Advantages of Quadlet

1. **Declarative**: Define containers in simple INI-style files
2. **Systemd Native**: Full integration with systemd (dependencies, logging, etc.)
3. **Auto-Updates**: Built-in support for automatic container updates
4. **Health Checks**: Native container health monitoring
5. **Simple**: No need to write complex systemd unit files
6. **Portable**: Easy to version control and deploy
7. **No Custom Users**: Container isolation handles security

## Security Considerations

- Container runs as non-root user (UID 1000 inside container)
- SELinux volume labels (`:Z`) for proper isolation
- Health checks for automatic restart on failure
- Read-only config volume mount
- No system user creation needed - container handles isolation

## Troubleshooting

### Service won't start

```bash
# Check systemd status
systemctl status opa

# Check Quadlet file
cat /etc/containers/systemd/opa.container

# Reload systemd to regenerate service from Quadlet
systemctl daemon-reload
systemctl start opa

# Check container directly
podman ps -a --filter name=opa
podman logs opa
```

### Permission issues

```bash
# Verify directory permissions
ls -la /var/lib/opa
ls -la /var/log/opa

# Ensure directories are writable
chmod 777 /var/lib/opa /var/log/opa

# Check SELinux labels
ls -laZ /var/lib/opa
ls -laZ /var/log/opa

# Relabel if needed
restorecon -R /var/lib/opa /var/log/opa
```

### Cannot connect to OPA

```bash
# Check if container is running
podman ps --filter name=opa

# Check container logs
podman logs opa

# Check port bindings
podman port opa

# Test from inside container
podman exec opa curl http://localhost:8181/health

# Check firewall
firewall-cmd --list-all
```

### Update container image

```bash
# Pull latest image
podman pull docker.io/openpolicyagent/opa:latest

# Restart service (will use new image)
systemctl restart opa

# Or enable auto-update
podman auto-update
```

## License

Apache-2.0

## Author Information

Created by Mitesh Sharma for RHDP at Red Hat.
