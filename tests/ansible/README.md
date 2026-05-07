# Ansible Test Suite for Packer Images

## Overview

This directory contains Ansible-based validation tests for Jenkins agent images built with Packer. The test suite validates that all required tools are installed with correct versions during the Packer provisioning process.

**Test Coverage**: 77 total tests across Linux and Windows platforms
- 23 cross-platform tests (common tools)
- 33 Linux-specific tests
- 21 Windows-specific tests (including OS-version-specific tests)

## Test Structure

```
tests/ansible/
├── ansible.cfg                    # Ansible configuration
├── inventory/
│   ├── local.yml                  # Localhost inventory
│   └── group_vars/
│       └── all.yml                # Expected tool versions
├── playbooks/
│   ├── test-common.yml            # Cross-platform tests (23 tests)
│   ├── test-linux.yml             # Linux tests (33 tests)
│   ├── test-windows.yml           # Windows tests (21 tests)
│   └── test-windows-2019.yml      # Windows 2019-specific tests
└── roles/
    └── validation/
        └── tasks/
            ├── common.yml         # Common validation tasks
            ├── linux.yml          # Linux validation tasks
            └── windows.yml        # Windows validation tasks
```

## Test Categories

### Common Tests (test-common.yml)
Validates cross-platform tools available on both Linux and Windows:
- AWS CLI, Azure CLI, Azure Copy
- Docker, Docker Compose, Docker Buildx
- Kubernetes tools (kubectl, helm, helmfile)
- HashiCorp tools (Terraform, Packer)
- GitHub CLI (gh)
- Git and Git LFS
- Hadolint, container-structure-test
- JQ, YQ, XQ
- jx-release-version
- UpdateCli, SOPS, Netlify Deploy

### Linux Tests (test-linux.yml)
Validates Linux-specific tools and configurations:
- asdf version manager
- Multiple JDK versions (8, 11, 17, 21, 25)
- Go, Ruby, Node.js
- Maven (with JAVA_HOME environment)
- Helm plugins (diff, git, secrets)
- Jenkins remoting JAR
- Python 3 and Launchable
- Ansible Core
- Datadog Agent
- Typos
- SSH Agent
- File permissions (/home/jenkins directory)

### Windows Tests (test-windows.yml)
Validates Windows-specific tools and configurations:
- Chocolatey package manager
- PowerShell and Windows PowerShell
- Multiple JDK versions (8, 11, 17, 21, 25)
- Maven (with MAVEN_HOME environment)
- .NET Framework 3.5
- Chromium browser
- Datadog Agent
- Make for Windows
- NuGet
- Ruby

### Windows Version-Specific Tests
- **test-windows-2019.yml**: Visual Studio 2019 MSBuild validation

## Version Management

Expected tool versions are defined in `inventory/group_vars/all.yml` and must be kept in sync with `provisioning/tools-versions.yml`.

**Example**:
```yaml
expected_versions:
  awscli: "2.34.43"
  docker: "29.4.2"
  golang: "1.26.2"
  # ... other versions
```

When updating tool versions in `provisioning/tools-versions.yml`, you must also update the corresponding entry in `inventory/group_vars/all.yml`.

## Running Tests Locally

### Prerequisites
- Ansible Core installed (version specified in `provisioning/tools-versions.yml`)
- Tools to be tested already installed on the system

### Linux Tests
```bash
cd tests/ansible

# Run common tests
ansible-playbook playbooks/test-common.yml

# Run Linux-specific tests
ansible-playbook playbooks/test-linux.yml

# Run all Linux tests
ansible-playbook playbooks/test-common.yml playbooks/test-linux.yml
```

### Windows Tests
```powershell
cd tests/ansible

# Run common tests
ansible-playbook playbooks/test-common.yml

# Run Windows-specific tests
ansible-playbook playbooks/test-windows.yml

# Run Windows version-specific tests (if applicable)
ansible-playbook playbooks/test-windows-2019.yml
```

## Integration with Packer

Tests are executed during Packer provisioning after all tools are installed.

### Linux Integration
Tests run as the `jenkins` user with asdf environment loaded:
```hcl
provisioner "shell" {
  execute_command  = "{{ .Vars }} sudo -E su - jenkins -c \"bash -eu '{{ .Path }}'\""
  environment_vars = local.provisioning_env_vars
  inline = [
    "source /home/jenkins/.asdf/asdf.sh",
    "cd /tmp/ansible",
    "ansible-playbook playbooks/test-common.yml",
    "ansible-playbook playbooks/test-linux.yml",
  ]
}
```

### Windows Integration
Tests run with elevated privileges:
```hcl
provisioner "powershell" {
  max_retries = 2
  environment_vars = local.provisioning_env_vars
  inline = [
    "cd C:/ansible",
    "ansible-playbook playbooks/test-common.yml",
    "ansible-playbook playbooks/test-windows.yml",
    "if (Test-Path playbooks/test-windows-${var.agent_os_version}.yml) {",
    "  ansible-playbook playbooks/test-windows-${var.agent_os_version}.yml",
    "}",
  ]
}
```

## Test Patterns

### Simple Version Check
```yaml
- name: Check AWS CLI version
  ansible.builtin.command: aws --version
  register: awscli_version
  changed_when: false
  failed_when: "'{{ expected_versions.awscli }}' not in awscli_version.stdout"
```

### Regex Pattern Matching
```yaml
- name: Check Helm plugins
  ansible.builtin.command: helm plugin list
  register: helm_plugins
  changed_when: false
  failed_when: >
    helm_plugins.stdout is not regex('diff.*3.15.6') or
    helm_plugins.stdout is not regex('helm-git.*1.5.2')
```

### Environment Variable Injection
```yaml
- name: Check Maven with JDK 21
  ansible.builtin.command: mvn -v
  environment:
    JAVA_HOME: /opt/jdk-21
  register: maven_version
  changed_when: false
  failed_when: "'3.9.15' not in maven_version.stdout"
```

### Negation Pattern (Expect Failure)
```yaml
- name: Verify no default Java in PATH
  ansible.builtin.command: java --version
  register: java_check
  changed_when: false
  failed_when: java_check.rc == 0
  ignore_errors: true
```

### File/Directory Validation
```yaml
- name: Check directory exists
  ansible.builtin.stat:
    path: /home/jenkins
  register: jenkins_home

- name: Validate directory properties
  ansible.builtin.assert:
    that:
      - jenkins_home.stat.exists
      - jenkins_home.stat.isdir
      - jenkins_home.stat.mode == '0750'
```

### Windows Registry Check
```yaml
- name: Check .NET Framework 3.5
  ansible.windows.win_reg_stat:
    path: HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP\v3.5
  register: dotnet35_reg

- name: Validate registry key exists
  ansible.builtin.assert:
    that:
      - dotnet35_reg.exists
```

## Troubleshooting

### Test Failures

**Check Ansible version**:
```bash
ansible-playbook --version
```

**Run with verbose output**:
```bash
ansible-playbook playbooks/test-common.yml -v
```

**Run specific task**:
```bash
ansible-playbook playbooks/test-common.yml --start-at-task="Check Docker version"
```

### Common Issues

**Issue**: `ERROR! 'environment' is not a valid attribute for a IncludeRole`  
**Solution**: Move `environment` block from task level to play level

**Issue**: Test expects tool in PATH but not found  
**Solution**: Verify tool installation and PATH environment in provisioning scripts

**Issue**: Version mismatch between expected and actual  
**Solution**: Update `inventory/group_vars/all.yml` to match `provisioning/tools-versions.yml`

## Migration from Goss

This test suite replaced the previous Goss-based validation (archived in `tests/goss-archive/`).

**Migration completed**: All 77 Goss tests migrated to Ansible with 100% parity
- goss-common.yaml (23 tests) → test-common.yml
- goss-linux.yaml (33 tests) → test-linux.yml
- goss-windows.yaml (21 tests) → test-windows.yml

**Benefits of Ansible**:
- Industry-standard tool with better Windows support
- More expressive test syntax
- Better error messages and debugging
- Aligns with infrastructure-as-code practices
- Native support for complex patterns (regex, environment variables, registry checks)

## Contributing

### Adding New Tests

1. **Identify the test category**: common, linux, or windows
2. **Add test to appropriate role task file**: `roles/validation/tasks/*.yml`
3. **Use idempotent patterns**: Always include `changed_when: false`
4. **Update version expectations**: Add to `inventory/group_vars/all.yml` if version-specific

### Updating Tool Versions

1. Update version in `provisioning/tools-versions.yml`
2. Update corresponding version in `inventory/group_vars/all.yml`
3. Test locally with `ansible-playbook` before committing
4. Verify in Packer build

## References

- [Ansible Documentation](https://docs.ansible.com/)
- [ansible.builtin modules](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html)
- [ansible.windows modules](https://docs.ansible.com/ansible/latest/collections/ansible/windows/index.html)
- [Packer Provisioners](https://www.packer.io/docs/provisioners)
