# asset_injector

Universal Ansible role for executing any Ansible module dynamically with parameter passing.

## Description

The `asset_injector` role provides a flexible and universal way to execute any Ansible module (core or collection) with dynamic parameter passing. Instead of writing separate playbooks or tasks for each operation, you can define all your infrastructure changes in a simple variable list, making it ideal for:

- Lab environment setup and configuration
- Host bootstrapping and initialization
- Day-2 operations and maintenance
- Configuration management across diverse systems
- Rapid prototyping and testing

## Key Features

- **Universal Module Support**: Execute ANY Ansible module without writing custom tasks
- **Dynamic Parameter Passing**: All module parameters are passed dynamically
- **Control Parameters**: Built-in support for become, when, retries, delegation, and more
- **Clean Separation**: Control logic separated from module parameters
- **Fully Idempotent**: Leverage Ansible's built-in idempotency for all operations
- **RHDP Compliant**: Follows AgnosticD v2 standards and patterns

## Requirements

- Ansible 2.9 or higher
- Target hosts must be accessible and properly configured in inventory
- Appropriate permissions for privileged operations (when using `become`)

## Role Variables

### Main Configuration

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `asset_injector_modules` | List of modules to execute with their parameters | `[]` | Yes |

### Module Definition Structure

Each module in `asset_injector_modules` requires:

#### Required Fields

| Field | Description | Example |
|-------|-------------|---------|
| `module` | Ansible module name (FQCN or short name) | `ansible.builtin.file`, `dnf`, `copy` |
| `description` | Human-readable description of the task | `Create application directory` |

#### Control Parameters (Optional)

| Parameter | Description | Default | Example |
|-----------|-------------|---------|---------|
| `become` | Enable privilege escalation | `false` | `true` |
| `become_user` | User to escalate to | `root` | `appuser` |
| `when` | Conditional execution | `true` | `ansible_os_family == "RedHat"` |
| `delegate_to` | Delegate task to another host | - | `bastion` |
| `tags` | Task tags for selective execution | - | `['config', 'security']` |
| `ignore_errors` | Continue on error | `false` | `true` |
| `changed_when` | Override change detection | - | `result.rc == 0` |
| `failed_when` | Override failure detection | - | `result.rc != 0` |
| `retries` | Number of retry attempts | - | `5` |
| `delay` | Delay between retries (seconds) | - | `10` |
| `until` | Retry until condition met | - | `result.status == 200` |
| `notify` | Handler to notify on change | - | `restart nginx` |
| `debug` | Show detailed execution results | `false` | `true` |

#### Module Parameters

All other fields are passed directly to the Ansible module. Refer to the specific module's documentation for available parameters.

## Dependencies

None

## Example Playbooks

### Example 1: Basic File and Service Management

```yaml
- hosts: webservers
  vars:
    asset_injector_modules:
      - description: Create web application directory
        module: ansible.builtin.file
        path: /var/www/myapp
        state: directory
        owner: nginx
        group: nginx
        mode: '0755'
        become: true

      - description: Deploy application configuration
        module: ansible.builtin.template
        src: templates/app.conf.j2
        dest: /etc/myapp/app.conf
        owner: root
        group: root
        mode: '0644'
        become: true
        notify: restart myapp

      - description: Ensure nginx is running
        module: ansible.builtin.service
        name: nginx
        state: started
        enabled: true
        become: true

  roles:
    - asset_injector
```

### Example 2: Package Installation and User Setup

```yaml
- hosts: appservers
  vars:
    app_user: myapp
    app_packages:
      - python3
      - python3-pip
      - git

    asset_injector_modules:
      - description: Install application dependencies
        module: ansible.builtin.dnf
        name: "{{ app_packages }}"
        state: present
        become: true

      - description: Create application user
        module: ansible.builtin.user
        name: "{{ app_user }}"
        state: present
        shell: /bin/bash
        home: /opt/{{ app_user }}
        create_home: true
        become: true

      - description: Clone application repository
        module: ansible.builtin.git
        repo: https://github.com/example/myapp.git
        dest: /opt/{{ app_user }}/src
        version: main
        become: true
        become_user: "{{ app_user }}"

  roles:
    - asset_injector
```

### Example 3: OpenShift Lab Configuration

```yaml
- hosts: bastion
  vars:
    student_name: lab-user
    openshift_api_url: https://api.ocp.example.com:6443
    openshift_token: sha256~xxxxxxxxxxxxxxx

    asset_injector_modules:
      - description: Remove cloud-init SSH config
        module: ansible.builtin.file
        path: /etc/ssh/sshd_config.d/50-cloud-init.conf
        state: absent
        become: true

      - description: Enable SSH password authentication
        module: ansible.builtin.blockinfile
        path: /etc/ssh/sshd_config
        state: present
        marker: "# {mark} ANSIBLE MANAGED - Lab SSH Configuration"
        block: |
          PasswordAuthentication yes
          PermitRootLogin no
        backup: true
        become: true

      - description: Restart SSH service
        module: ansible.builtin.service
        name: sshd
        state: restarted
        become: true

      - description: OpenShift login as student
        module: ansible.builtin.command
        cmd: "/usr/bin/oc login {{ openshift_api_url }} --token={{ openshift_token }} --insecure-skip-tls-verify=true"
        become: true
        become_user: "{{ student_name }}"

      - description: Download AAP manifest
        module: ansible.builtin.get_url
        url: https://example.com/aap_manifest.zip
        dest: /home/{{ student_name }}/aap_manifest.zip
        owner: "{{ student_name }}"
        group: "{{ student_name }}"
        mode: '0644'
        become: true

  roles:
    - asset_injector
```

### Example 4: Advanced Usage with Control Parameters

```yaml
- hosts: production
  vars:
    asset_injector_modules:
      # Task with retry logic for API health check
      - description: Wait for API to be healthy
        module: ansible.builtin.uri
        url: https://api.example.com/health
        method: GET
        status_code: 200
        retries: 10
        delay: 5
        until: result.status == 200
        ignore_errors: false
        debug: true

      # Conditional execution based on OS family
      - description: Install RHEL-specific packages
        module: ansible.builtin.dnf
        name:
          - subscription-manager
          - redhat-lsb-core
        state: present
        become: true
        when: ansible_os_family == "RedHat"

      # Delegated task execution
      - description: Deploy Kubernetes manifest from bastion
        module: ansible.builtin.command
        cmd: kubectl apply -f /tmp/deployment.yml
        delegate_to: bastion
        become: true

      # Custom change detection
      - description: Check application version
        module: ansible.builtin.shell
        cmd: /opt/myapp/bin/myapp --version
        register: app_version
        changed_when: false
        failed_when: app_version.rc != 0

  roles:
    - asset_injector
```

### Example 5: Complete Application Deployment

```yaml
- hosts: appservers
  vars:
    app_name: myapp
    app_version: 1.0.0
    app_user: appuser
    app_port: 8080

    asset_injector_modules:
      # User and group setup
      - description: Create application group
        module: ansible.builtin.group
        name: "{{ app_user }}"
        state: present
        become: true

      - description: Create application user
        module: ansible.builtin.user
        name: "{{ app_user }}"
        group: "{{ app_user }}"
        home: /opt/{{ app_name }}
        shell: /bin/bash
        create_home: true
        become: true

      # Directory structure
      - description: Create application directories
        module: ansible.builtin.file
        path: "{{ item }}"
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_user }}"
        mode: '0755'
        become: true
        with_items:
          - /opt/{{ app_name }}
          - /opt/{{ app_name }}/bin
          - /opt/{{ app_name }}/config
          - /var/log/{{ app_name }}

      # Package installation
      - description: Install runtime dependencies
        module: ansible.builtin.dnf
        name:
          - python3
          - python3-pip
          - nginx
        state: present
        become: true

      # Application deployment
      - description: Download application binary
        module: ansible.builtin.get_url
        url: https://releases.example.com/{{ app_name }}/{{ app_version }}/{{ app_name }}.tar.gz
        dest: /tmp/{{ app_name }}.tar.gz
        checksum: sha256:abc123...
        become: true

      - description: Extract application
        module: ansible.builtin.unarchive
        src: /tmp/{{ app_name }}.tar.gz
        dest: /opt/{{ app_name }}/
        remote_src: true
        owner: "{{ app_user }}"
        group: "{{ app_user }}"
        become: true

      # Configuration
      - description: Deploy application configuration
        module: ansible.builtin.template
        src: templates/{{ app_name }}.conf.j2
        dest: /opt/{{ app_name }}/config/app.conf
        owner: "{{ app_user }}"
        group: "{{ app_user }}"
        mode: '0644'
        become: true

      - description: Create systemd service file
        module: ansible.builtin.template
        src: templates/{{ app_name }}.service.j2
        dest: /etc/systemd/system/{{ app_name }}.service
        owner: root
        group: root
        mode: '0644'
        become: true

      # Service management
      - description: Reload systemd
        module: ansible.builtin.systemd
        daemon_reload: true
        become: true

      - description: Enable and start application service
        module: ansible.builtin.service
        name: "{{ app_name }}"
        state: started
        enabled: true
        become: true

      # Firewall configuration
      - description: Open application port in firewall
        module: ansible.posix.firewalld
        port: "{{ app_port }}/tcp"
        permanent: true
        state: enabled
        immediate: true
        become: true

  roles:
    - asset_injector
```

## Supported Modules

This role supports **ALL** Ansible modules including:

### Core Modules (ansible.builtin)
- File operations: `file`, `copy`, `template`, `lineinfile`, `blockinfile`
- Package management: `dnf`, `yum`, `apt`, `pip`, `package`
- Service management: `service`, `systemd`
- User management: `user`, `group`
- Command execution: `command`, `shell`, `script`
- File transfer: `get_url`, `unarchive`, `fetch`
- Source control: `git`, `subversion`
- System: `mount`, `cron`, `authorized_key`, `hostname`, `timezone`
- And many more...

### Collection Modules
- `community.general.*`
- `ansible.posix.*`
- `community.crypto.*`
- `community.docker.*`
- `kubernetes.core.*`
- Any installed collection module

Simply specify the module FQCN or short name in the `module` field.

## How It Works

1. The role iterates through `asset_injector_modules` list
2. For each item, it separates control parameters from module parameters
3. Control parameters (become, when, retries, etc.) are applied to the task
4. Module parameters are passed to the specified Ansible module using the `action` plugin
5. The module executes with all its parameters dynamically
6. Results are captured and optionally displayed (when `debug: true`)

## Best Practices

1. **Use FQCN**: Always use fully qualified collection names (e.g., `ansible.builtin.file`) for clarity
2. **Descriptive Names**: Write clear, concise descriptions for each task
3. **Idempotency**: Leverage module idempotency - specify desired state, not actions
4. **Error Handling**: Use `ignore_errors`, `failed_when`, and `retries` appropriately
5. **Privilege Escalation**: Only use `become: true` when necessary
6. **Conditionals**: Use `when` to skip unnecessary tasks
7. **Organization**: Group related tasks together in the list
8. **Documentation**: Comment complex operations in your vars file

## Troubleshooting

### Task fails with "module not found"
- Ensure the module name is correct (check Ansible documentation)
- Verify required collections are installed
- Use FQCN to avoid ambiguity

### Parameters not being passed correctly
- Check that control parameters are in the exclude list
- Verify parameter names match module documentation
- Use `debug: true` to see what's being passed

### Privilege escalation issues
- Ensure `become: true` is set when needed
- Verify `become_user` has necessary permissions
- Check sudoers configuration on target hosts

## Integration with RHDP

This role is designed to work seamlessly with Red Hat Demo Platform (RHDP) environments:

- Follows AgnosticD v2 patterns
- Compatible with CNV pools and namespace APIs
- Supports bastion delegation for OpenShift operations
- Integrates with RHDP GitOps workflows
- Ideal for lab environment bootstrapping

## Migration from Module-Specific Files

If you were using the old approach with separate task files per module:

**Old approach**:
```yaml
- module: file
  path: /tmp/test
  state: absent
```

**New universal approach** (same configuration):
```yaml
- description: Remove test file
  module: ansible.builtin.file
  path: /tmp/test
  state: absent
```

Simply add a `description` field and optionally use FQCN for the module name.

## License

GPL-3.0-or-later

## Author

Mitesh Sharma <mitsharm@redhat.com>
Red Hat Demo Platform (RHDP)

## Contributing

Issues and pull requests welcome. Please follow RHDP development standards.

## See Also

- [Ansible Module Index](https://docs.ansible.com/ansible/latest/collections/index_module.html)
- [AgnosticD v2 Documentation](https://github.com/agnosticd/)
- [RHDP Best Practices](https://github.com/rhpds/)
