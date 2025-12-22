# aap_api

Ansible role to manage Ansible Automation Platform (AAP) Controller resources via REST API.

## Description

This role provides a flexible way to interact with AAP Controller's REST API to manage various resources. Currently supports creating credentials, with extensibility for other resource types.

## Requirements

- Ansible 2.9 or higher
- Access to AAP Controller instance
- Valid AAP Controller credentials with appropriate permissions

## Role Variables

### Connection Settings

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `aap_api_controller_url` | AAP Controller base URL | `https://aap.example.com` | Yes |
| `aap_api_controller_username` | AAP Controller username | `<CHANGEME>` | Yes |
| `aap_api_controller_password` | AAP Controller password | `<CHANGEME>` | Yes |
| `aap_api_controller_verify_ssl` | Verify SSL certificates | `false` | No |

### Endpoint Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `aap_api_endpoints` | List of API endpoints to process | See example below |

## Dependencies

None

## Example Playbook

### Basic Usage - Create OpenShift Credential

```yaml
- hosts: localhost
  vars:
    aap_api_controller_url: https://controller.example.com
    aap_api_controller_username: admin
    aap_api_controller_password: secret123
    aap_api_controller_verify_ssl: false

    openshift_api_url: https://api.openshift.example.com:6443
    openshift_api_token: sha256~xxxxxxxxxxxxxxxxxxxxx

    aap_api_endpoints:
      - type: credential
        name: my-openshift-cluster
        api: /api/v2/
        description: Production OpenShift cluster credential
        organization: Default
        credential_type: OpenShift or Kubernetes API Bearer Token
        inputs:
          host: "{{ openshift_api_url }}"
          bearer_token: "{{ openshift_api_token }}"
          verify_ssl: false

  roles:
    - aap_api
```

### Multiple Credentials

```yaml
- hosts: localhost
  vars:
    aap_api_controller_url: https://controller.example.com
    aap_api_controller_username: admin
    aap_api_controller_password: secret123

    aap_api_endpoints:
      - type: credential
        name: dev-openshift
        description: Development OpenShift cluster
        organization: Engineering
        credential_type: OpenShift or Kubernetes API Bearer Token
        inputs:
          host: https://api.dev.example.com:6443
          bearer_token: "{{ dev_token }}"
          verify_ssl: true

      - type: credential
        name: prod-openshift
        description: Production OpenShift cluster
        organization: Operations
        credential_type: OpenShift or Kubernetes API Bearer Token
        inputs:
          host: https://api.prod.example.com:6443
          bearer_token: "{{ prod_token }}"
          verify_ssl: true

  roles:
    - aap_api
```

## Endpoint Configuration Format

Each endpoint in `aap_api_endpoints` supports:

| Field | Description | Required |
|-------|-------------|----------|
| `type` | Resource type (currently only 'credential' supported) | Yes |
| `name` | Name of the resource (also used as description fallback) | Yes |
| `api` | API path (default: `/api/v2/`) | No |
| `description` | Description of the resource (defaults to `name` if not provided) | No |
| `organization` | Organization name in AAP | No (default: 'Default') |
| `credential_type` | Credential type name | Yes for credentials |
| `inputs` | Dictionary of credential inputs | Yes for credentials |

## Supported Credential Types

Common credential types in AAP Controller:

- `Machine` - SSH credentials for machines
- `Source Control` - Git, SVN credentials
- `Vault` - Ansible Vault password
- `Network` - Network device credentials
- `Amazon Web Services` - AWS access credentials
- `OpenShift or Kubernetes API Bearer Token` - OpenShift/K8s credentials
- `Red Hat Ansible Automation Platform` - AAP credentials
- `Container Registry` - Container registry credentials

Run this to list all available credential types in your AAP instance:

```bash
curl -k -u admin:password https://controller.example.com/api/v2/credential_types/ | jq '.results[] | {name, kind}'
```

## Error Handling

The role includes validation for:

- Organization existence
- Credential type existence
- Empty API responses

If any validation fails, the role will fail with a descriptive error message.

## Extending the Role

To add support for other AAP resources:

1. Create a new task file: `tasks/<resource_type>.yml`
2. Follow the pattern in `tasks/credential.yml`
3. Add appropriate defaults in `defaults/main.yml`
4. Update this README with examples

## License

GPL-3.0-or-later

## Author

Mitesh, The Mouse <mitsharm@redhat.com>
Red Hat

## Contributing

Issues and pull requests welcome. Please follow RHDP development standards.
