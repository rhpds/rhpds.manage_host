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
| `aap_api_job_launches` | List of job templates to launch | `[]` (empty list) |

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

### Example 3: Create Hosts in Inventory

```yaml
- hosts: localhost
  vars:
    aap_api_controller_url: https://controller.example.com
    aap_api_controller_username: admin
    aap_api_controller_password: secret123

    aap_api_endpoints:
      # Linux host
      - name: web-server-01
        endpoint: hosts
        description: Production web server
        inventory: production-inventory
        attributes:
          variables: |
            ---
            ansible_host: 192.168.1.10
            ansible_connection: ssh
            ansible_user: ansible
          enabled: true

      # Windows host
      - name: windows-server
        endpoint: hosts
        description: Windows Server 2019
        inventory: production-inventory
        attributes:
          variables: |
            ---
            ansible_host: 192.168.1.20
            ansible_connection: psrp
            ansible_psrp_auth: negotiate
            ansible_psrp_cert_validation: ignore
          enabled: true

  roles:
    - aap_api
```

### Example 4: AWS and Machine Credentials

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

### Example 5: Launch Job Templates

```yaml
- hosts: localhost
  vars:
    aap_api_controller_url: https://controller.example.com
    aap_api_controller_username: admin
    aap_api_controller_password: secret123

    # First, create/update resources
    aap_api_endpoints:
      - name: deploy-app
        endpoint: job_templates
        description: Deploy application to production
        organization: Engineering
        project: ansible-playbooks
        inventory: prod-servers
        attributes:
          playbook: deploy.yml
          job_type: run

    # Then, launch the job template
    aap_api_job_launches:
      # Simple launch
      - name: deploy-app
        organization: Engineering

  roles:
    - aap_api
```

### Example 6: Launch Job Template with Extra Variables

```yaml
- hosts: localhost
  vars:
    aap_api_controller_url: https://controller.example.com
    aap_api_controller_username: admin
    aap_api_controller_password: secret123

    aap_api_job_launches:
      # Launch with extra variables
      - name: deploy-app
        organization: Engineering
        extra_vars:
          environment: production
          version: v2.1.0
          enable_feature_x: true

  roles:
    - aap_api
```

### Example 7: Launch and Wait for Completion

```yaml
- hosts: localhost
  vars:
    aap_api_controller_url: https://controller.example.com
    aap_api_controller_username: admin
    aap_api_controller_password: secret123

    aap_api_job_launches:
      # Launch and wait for job to complete
      - name: deploy-app
        organization: Engineering
        wait: true
        wait_retries: 60    # Check up to 60 times
        wait_delay: 10      # Wait 10 seconds between checks
        extra_vars:
          environment: staging
          version: v2.1.0

  roles:
    - aap_api
```

### Example 8: Launch with Limit and Tags

```yaml
- hosts: localhost
  vars:
    aap_api_controller_url: https://controller.example.com
    aap_api_controller_username: admin
    aap_api_controller_password: secret123

    aap_api_job_launches:
      # Launch with host limit and specific tags
      - name: configure-servers
        organization: Engineering
        limit: web-servers
        tags: nginx,ssl,firewall
        skip_tags: debug,testing

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
| `organization` | Organization name (resolved to ID) | - | Most resources (NOT hosts) |
| `credential_type` | Credential type name (resolved to ID) | - | Credentials only |
| `project` | Project name (resolved to ID) | - | Job templates |
| `inventory` | Inventory name (resolved to ID) | - | Job templates, hosts |
| `attributes` | Additional resource-specific attributes | `{}` | All resources |

### Important Notes for Hosts

When creating hosts (`endpoint: hosts`):
- **DO NOT** include `organization` field - hosts belong to inventories, not directly to organizations
- **MUST** include `inventory` field - specifies which inventory the host belongs to
- Common attributes for hosts:
  - `variables` - Host-specific Ansible variables (YAML string)
  - `enabled` - Whether the host is enabled (boolean)
  - `instance_id` - Cloud provider instance ID (optional)

Example host configuration:
```yaml
- name: my-host
  endpoint: hosts
  description: My server
  inventory: my-inventory  # Required for hosts
  # organization: Default  # Do NOT include for hosts
  attributes:
    variables: |
      ---
      ansible_host: 10.0.0.1
      ansible_user: admin
    enabled: true
```

### How It Works

1. **name** and **endpoint** are always required
2. The role automatically resolves references (organization, credential_type, project, inventory) to their IDs
3. All other resource-specific fields go into the **attributes** dictionary
4. The role checks if the resource exists:
   - If exists: Updates it (PATCH)
   - If not: Creates it (POST)
5. Operations are fully idempotent

## Job Launch Configuration Format

Each job template launch in `aap_api_job_launches` uses this structure:

### Required Fields

| Field | Description | Example |
|-------|-------------|---------|
| `name` | Name of the job template to launch | `deploy-application` |

### Optional Fields

| Field | Description | Default | Example |
|-------|-------------|---------|---------|
| `organization` | Organization name (helps locate job template) | - | `Engineering` |
| `api` | API path | `/api/controller/v2/` | `/api/controller/v2/` |
| `extra_vars` | Additional variables for the job | `{}` | `{environment: prod, version: v1.0}` |
| `limit` | Limit job run to specific hosts | - | `web-servers` |
| `tags` | Only run plays and tasks tagged with these values | - | `deploy,config` |
| `skip_tags` | Skip plays and tasks tagged with these values | - | `debug,test` |
| `inventory_id` | Inventory ID to use (override job template default) | - | `5` |
| `credential_ids` | List of credential IDs to use | `[]` | `[1, 2, 3]` |
| `wait` | Wait for job completion before continuing | `false` | `true` |
| `wait_retries` | Number of times to check job status | `60` | `120` |
| `wait_delay` | Seconds to wait between status checks | `10` | `15` |

### How Job Launches Work

1. **name** is always required
2. The role finds the job template by name (optionally filtered by organization)
3. Launches the job template with specified parameters
4. If `wait: true`, polls the job status until completion
5. Fails if `wait: true` and the job fails/errors/is canceled
6. Returns job ID and URL for monitoring

### Job Launch Examples

**Simple launch:**
```yaml
aap_api_job_launches:
  - name: my-job-template
    organization: Default
```

**Launch with variables:**
```yaml
aap_api_job_launches:
  - name: deploy-app
    organization: Engineering
    extra_vars:
      env: production
      debug: false
```

**Launch and wait:**
```yaml
aap_api_job_launches:
  - name: provision-infrastructure
    organization: Operations
    wait: true
    wait_retries: 120
    wait_delay: 15
    extra_vars:
      region: us-east-1
```

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
| Hosts | `hosts` | Individual hosts within inventories |
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
