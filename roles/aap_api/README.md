# aap_api

Ansible role to manage Ansible Automation Platform (AAP) Controller resources via REST API.

## Description

This role provides a flexible and universal way to interact with AAP Controller's REST API to manage any type of resource including credentials, projects, inventories, job templates, organizations, teams, and more. The role automatically detects if a resource exists and updates it, or creates it if it doesn't exist (idempotent operations).

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

### Example 1: Create/Update OpenShift Credential

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
      - name: my-openshift-cluster
        endpoint: credentials
        description: Production OpenShift cluster credential
        organization: Default
        credential_type: OpenShift or Kubernetes API Bearer Token
        attributes:
          inputs:
            host: "{{ openshift_api_url }}"
            bearer_token: "{{ openshift_api_token }}"
            verify_ssl: false

  roles:
    - aap_api
```

### Example 2: Create/Update Multiple Resources

```yaml
- hosts: localhost
  vars:
    aap_api_controller_url: https://controller.example.com
    aap_api_controller_username: admin
    aap_api_controller_password: secret123

    aap_api_endpoints:
      # Create organization
      - name: Engineering
        endpoint: organizations
        description: Engineering team organization
        attributes:
          max_hosts: 100

      # Create credential
      - name: dev-openshift
        endpoint: credentials
        description: Development OpenShift cluster
        organization: Engineering
        credential_type: OpenShift or Kubernetes API Bearer Token
        attributes:
          inputs:
            host: https://api.dev.example.com:6443
            bearer_token: "{{ dev_token }}"
            verify_ssl: true

      # Create project
      - name: ansible-playbooks
        endpoint: projects
        description: Ansible automation playbooks
        organization: Engineering
        attributes:
          scm_type: git
          scm_url: https://github.com/example/playbooks.git
          scm_branch: main
          scm_update_on_launch: true

      # Create inventory
      - name: dev-servers
        endpoint: inventories
        description: Development environment servers
        organization: Engineering
        attributes:
          variables: |
            ---
            ansible_connection: ssh
            ansible_user: ansible

      # Create job template
      - name: deploy-app
        endpoint: job_templates
        description: Deploy application to dev
        organization: Engineering
        project: ansible-playbooks
        inventory: dev-servers
        attributes:
          playbook: deploy.yml
          job_type: run
          ask_variables_on_launch: true

  roles:
    - aap_api
```

### Example 3: AWS and Machine Credentials

```yaml
- hosts: localhost
  vars:
    aap_api_controller_url: https://controller.example.com
    aap_api_controller_username: admin
    aap_api_controller_password: secret123

    aap_api_endpoints:
      # SSH credential for Linux servers
      - name: linux-ssh-key
        endpoint: credentials
        description: SSH key for Linux servers
        organization: Default
        credential_type: Machine
        attributes:
          inputs:
            username: ansible
            ssh_key_data: "{{ lookup('file', '~/.ssh/id_rsa') }}"
            become_method: sudo

      # AWS credential
      - name: aws-production
        endpoint: credentials
        description: AWS production account
        organization: Default
        credential_type: Amazon Web Services
        attributes:
          inputs:
            username: "{{ aws_access_key_id }}"
            password: "{{ aws_secret_access_key }}"

  roles:
    - aap_api
```

## Endpoint Configuration Format

Each endpoint in `aap_api_endpoints` uses a universal structure:

### Required Fields

| Field | Description | Example |
|-------|-------------|---------|
| `name` | Name of the resource | `my-openshift-cluster` |
| `endpoint` | AAP API endpoint type | `credentials`, `projects`, `inventories`, `job_templates`, `organizations`, `teams` |

### Optional Fields

| Field | Description | Default | Used For |
|-------|-------------|---------|----------|
| `api` | API path | `/api/controller/v2/` | All resources |
| `description` | Description of the resource | `name` value | All resources |
| `organization` | Organization name (resolved to ID) | - | Most resources |
| `credential_type` | Credential type name (resolved to ID) | - | Credentials only |
| `project` | Project name (resolved to ID) | - | Job templates |
| `inventory` | Inventory name (resolved to ID) | - | Job templates |
| `attributes` | Additional resource-specific attributes | `{}` | All resources |

### How It Works

1. **name** and **endpoint** are always required
2. The role automatically resolves references (organization, credential_type, project, inventory) to their IDs
3. All other resource-specific fields go into the **attributes** dictionary
4. The role checks if the resource exists:
   - If exists: Updates it (PATCH)
   - If not: Creates it (POST)
5. Operations are fully idempotent

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

## Supported AAP Resources

This role uses a universal approach and supports ALL AAP Controller API endpoints, including:

### Common Resources

| Resource Type | Endpoint Value | Description |
|--------------|---------------|-------------|
| Credentials | `credentials` | Machine, SSH, cloud provider, API tokens, etc. |
| Projects | `projects` | Git/SVN repositories containing playbooks |
| Inventories | `inventories` | Host inventories |
| Job Templates | `job_templates` | Playbook execution templates |
| Workflow Job Templates | `workflow_job_templates` | Multi-job workflows |
| Organizations | `organizations` | Organizational units |
| Teams | `teams` | User groups within organizations |
| Users | `users` | AAP users |
| Schedules | `schedules` | Scheduled job runs |
| Notification Templates | `notification_templates` | Notification integrations |
| Execution Environments | `execution_environments` | Container images for job execution |

### Advanced Resources

| Resource Type | Endpoint Value | Description |
|--------------|---------------|-------------|
| Inventory Sources | `inventory_sources` | Dynamic inventory sources |
| Groups | `groups` | Inventory groups |
| Hosts | `hosts` | Individual hosts |
| Applications | `applications` | OAuth2 applications |
| Tokens | `tokens` | API tokens |

### No Extension Needed

The role is designed to work with ANY AAP Controller endpoint without modification. Simply:

1. Identify the endpoint name from AAP API (e.g., `/api/controller/v2/credentials/`)
2. Use the endpoint name (without slashes) in your configuration
3. Add resource-specific fields in the `attributes` dictionary
4. The role handles the rest automatically

## License

GPL-3.0-or-later

## Author

Mitesh, The Mouse <mitsharm@redhat.com>
Red Hat

## Contributing

Issues and pull requests welcome. Please follow RHDP development standards.
