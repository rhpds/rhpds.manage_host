# SSAP Setup Role

This Ansible role automates the setup and configuration of SSAP (Self-Service Automation Portal) with Ansible Automation Platform (AAP) integration.

## Description

The `ssap_setup` role performs the following tasks:

1. **OAuth Application Creation** - Creates an OAuth application in AAP Controller for SSAP authentication
2. **Admin Token Generation** - Creates a personal access token for the admin user
3. **Container Image Setup** - Logs into Red Hat registry and pulls required container images:
   - PostgreSQL 15
   - Ansible Dev Tools
   - Red Hat Developer Hub (RHDH)
4. **Podman Secrets Management** - Creates podman secrets for OAuth client secret and AAP token
5. **Portal Configuration** - Generates the portal environment configuration file

## Requirements

- Ansible 2.9 or higher
- Access to Red Hat Container Registry (registry.redhat.io)
- AAP Controller with admin credentials
- Podman installed on the target host
- OpenShift cluster with accessible subdomain

## Role Variables

### Required Variables

```yaml
subdomain: "wqjft.apps.ocpvdev01.rhdp.net"
registry_credentials_username: "your-rh-username@redhat.com"
registry_credentials_password: "your-rh-password"
```

### Optional Variables (with defaults)

```yaml
# AAP Configuration
aap_image_admin_password: "ansible123!"
ssap_aap_username: admin

# OAuth Application Settings
ssap_aap_oauth_application_name_prefix: "Selfservice"
ssap_aap_oauth_organization_id: 1
ssap_aap_oauth_client_type: "confidential"
ssap_aap_oauth_grant_type: "authorization-code"

# Container Images
ssap_postgresql_image: "registry.redhat.io/rhel9/postgresql-15:latest"
ssap_ansible_devtools_image: "registry.redhat.io/ansible-automation-platform-26/ansible-dev-tools-rhel9:latest"
ssap_rhdh_image: "registry.redhat.io/rhdh/rhdh-hub-rhel9:1.8"

# Portal Configuration
ssap_portal_environment: "production"
ssap_use_external_postgres: false
ssap_postgresql_host: "portal-postgres"
ssap_postgresql_port: 5432
ssap_postgresql_database: "portal_db"
ssap_postgresql_user: "portal_user"
ssap_portal_host_port: 443
ssap_aap_check_ssl: false

# Output Paths
ssap_credentials_dir: "/tmp"
ssap_podman_authfile: "/etc/containers/auth.json"
ssap_podman_secret_oauth_name: "portal_aap_oauth_client_secret"
ssap_podman_secret_token_name: "portal_aap_token"
```

## Dependencies

None.

## Example Playbook

```yaml
---
- name: Setup SSAP Portal
  hosts: bastion
  become: true

  vars:
    subdomain: "wqjft.apps.ocpvdev01.rhdp.net"
    registry_credentials_username: "mitsharm@redhat.com"
    registry_credentials_password: "your-password-here"
    aap_image_admin_password: "ansible123!"

  roles:
    - rhpds.manage_host.ssap_setup
```

## Example with Custom Variables

```yaml
---
- name: Setup SSAP Portal with Custom Configuration
  hosts: bastion
  become: true

  vars:
    subdomain: "custom.apps.cluster.example.com"
    registry_credentials_username: "user@redhat.com"
    registry_credentials_password: "secure-password"
    aap_image_admin_password: "custom-admin-pass"

    # Custom portal configuration
    ssap_portal_environment: "development"
    ssap_credentials_dir: "/var/lib/ssap"
    ssap_postgresql_host: "external-postgres.example.com"
    ssap_use_external_postgres: true

  roles:
    - rhpds.manage_host.ssap_setup
```

## Generated Files

The role creates the following files in `{{ ssap_credentials_dir }}` (default: `/tmp`):

- `client_id.txt` - OAuth application client ID
- `client_secret.txt` - OAuth application client secret
- `application_id.txt` - OAuth application ID
- `aap_token.txt` - AAP admin personal access token
- `oauth_application.yaml` - Full OAuth application response (YAML)
- `aap_token.yaml` - Full token response (YAML)
- `portal.env` - Portal environment configuration file

## Podman Secrets Created

- `portal_aap_oauth_client_secret` - OAuth client secret for SSAP
- `portal_aap_token` - AAP admin token for API access

## Testing the Setup

After running the role, you can test the AAP token:

```bash
# Read the token
AAP_TOKEN=$(cat /tmp/aap_token.txt)

# Test API access
curl -H "Authorization: Bearer ${AAP_TOKEN}" \
     https://controller-${SUBDOMAIN}/api/gateway/v1/me/
```

## Portal URLs

After setup, the following URLs will be available:

- **SSAP Portal**: `https://ssap-portal-{{ subdomain }}`
- **AAP Controller**: `https://controller-{{ subdomain }}`
- **OAuth Redirect**: `https://ssap-portal-{{ subdomain }}/api/auth/rhaap/handler/frame`

## Troubleshooting

### Registry Login Fails

Ensure your Red Hat account has access to the container registry and that credentials are correct.

```bash
# Manually test login
podman login registry.redhat.io -u your-username
```

### OAuth Application Creation Fails

Check that:
- AAP Controller is accessible at `https://controller-{{ subdomain }}`
- Admin credentials are correct
- The organization ID exists (default: 1)

### Podman Secret Already Exists

The role automatically removes existing secrets with the same name before creating new ones.

## License

Apache-2.0

## Author Information

Created by the RHDP Team at Red Hat.
