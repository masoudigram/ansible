# Cisco Backup and Version Control Using Ansible

A collection of Ansible playbooks for network operations, infrastructure automation, configuration management, and operational reliability.

This repository is designed to automate repetitive network administration tasks while maintaining configuration consistency, traceability, and operational safety across network infrastructure.

## Overview

The playbooks in this repository support network engineering and operations workflows, including:

- **Cisco network automation** — Configuration collection and device management using Ansible network modules.
- **Configuration backup** — Automated collection and preservation of device configurations.
- **Version-controlled backups** — Git-based configuration history, change tracking, and recovery.
- **AWX integration** — Execution through scheduled jobs, inventories, credentials, and job templates.
- **Operational automation** — Reusable workflows for network maintenance and routine administration.

The repository is intended to evolve as additional automation workflows are developed and validated.

## Architecture

The configuration backup workflow integrates Ansible, AWX, Cisco network devices, and GitLab.

```text
                    +----------------------+
                    |      AWX / Ansible   |
                    |   Job Template       |
                    +----------+-----------+
                               |
                               v
                    +----------------------+
                    |  Cisco Network       |
                    |  Devices             |
                    +----------+-----------+
                               |
                               | Network CLI
                               v
                    +----------------------+
                    | cisco.ios.ios_config |
                    | Running Config       |
                    | Backup Collection    |
                    +----------+-----------+
                               |
                 +-------------+-------------+
                 |                           |
                 v                           v
       +-------------------+      +----------------------+
       | Local Backup      |      | Normalization        |
       | Timestamped Files |      | Remove Volatile      |
       |                   |      | Metadata             |
       +-------------------+      +----------+-----------+
                                             |
                                             v
                                  +----------------------+
                                  | Git Working Tree     |
                                  | Stable Device Files  |
                                  +----------+-----------+
                                             |
                                             v
                                  +----------------------+
                                  | Change Detection     |
                                  | Git Diff             |
                                  +----------+-----------+
                                             |
                                    Changes detected?
                                      /          \
                                    Yes           No
                                     |             |
                                     v             v
                              +------------+  +------------+
                              | Commit and |  | No Commit  |
                              | Push to    |  | Required   |
                              | main       |  +------------+
                              +------------+
```

## Repository Structure

A recommended directory structure is:

```text
.
├── README.md
├── backups/
│   └── cisco/
│       ├── <device-1>.config
│       ├── <device-2>.config
│       └── <device-n>.config
└── playbooks/
    ├── cisco-backup.yml
    └── ...
```

The actual structure may differ depending on how playbooks and backup artifacts are organized in your AWX project.

### Configuration Backup Storage

The backup workflow maintains two distinct representations of each device configuration.

| Destination | Filename | Purpose |
|---|---|---|
| Local backup directory | `<hostname>-<timestamp>.config` | Preserves individual backup snapshots |
| Git repository | `<hostname>.config` | Maintains a stable, version-controlled configuration |

The Git repository contains one current configuration file per device. Git history preserves previous committed versions.

## Cisco Configuration Backup

Cisco configuration backups use the `cisco.ios.ios_config` module.

Example:

```yaml
- name: Cisco IOS Configuration Backup
  hosts: cisco
  gather_facts: false
  connection: ansible.netcommon.network_cli

  vars:
    ansible_network_os: cisco.ios.ios

  tasks:
    - name: Back up running configuration
      cisco.ios.ios_config:
        backup: true
        backup_options:
          dir_path: /var/lib/awx/projects/output/cisco/
      register: backup_output

    - name: Display backup result
      ansible.builtin.debug:
        msg: "Backup saved: {{ backup_output.backup_path }}"
```

This example illustrates the configuration collection mechanism. The complete production workflow additionally validates the generated file, normalizes volatile metadata, and integrates with Git.

### Backup Processing

The backup workflow follows these steps:

1. Connect to the Cisco device using Ansible's network CLI connection.
2. Collect the running configuration.
3. Validate the generated backup file.
4. Preserve a timestamped local copy.
5. Normalize volatile metadata for version control.
6. Compare the normalized configuration against the existing Git version.
7. Commit and push changes when necessary.

### Metadata Normalization

Cisco configuration output can include generated metadata that changes even when the actual configuration remains unchanged.

Examples include:

```text
Building configuration...

Current configuration : 65593 bytes
!
! Last configuration change at 17:58:41 UTC Sat Oct 24 2026 by mas_ramezani
! NVRAM config last updated at 18:05:26 UTC Sat Oct 24 2026 by mas_ramezani
!Command: show running-config
!Running configuration last done at: Wed Oct 14 11:44:03 2026
!Time: Tue Oct 27 12:47:58 2026
```

The normalization stage removes the explicitly configured volatile metadata before Git comparison.

This prevents unnecessary commits caused by timestamps, configuration-size changes, and other generated header information.

**Important:** Normalization rules should be reviewed for each supported Cisco platform. Only known non-semantic metadata should be removed.

## Git-Based Configuration Versioning

The backup workflow uses Git to maintain configuration history.

Repository:

```text
git@gitlab.example.local:ip-team/awx-ansible.git
```

Target branch:

```text
main
```

### Change Detection

The workflow compares normalized configurations with the versions tracked in Git.

| Condition | Expected behavior |
|---|---|
| Device is backed up for the first time | Create a new configuration file and commit it |
| Configuration is unchanged | Do not create a commit |
| Configuration changes | Update the existing file and commit the change |
| Only normalized metadata changes | Do not create a commit |
| Configuration collection fails | Do not update that device's Git backup |

### Commit History

Git provides a history of committed configuration changes.

Inspect recent backup commits:

```bash
git log --oneline -- backups/cisco/
```

Review a specific device's history:

```bash
git log --follow -- backups/cisco/<hostname>.config
```

Compare the two most recent versions of a device configuration:

```bash
git diff HEAD~1 HEAD -- backups/cisco/<hostname>.config
```

Restore a configuration from a previous commit for inspection:

```bash
git show <commit-id>:backups/cisco/<hostname>.config
```

These commands help network engineers investigate configuration changes and identify when a change entered version control.

## Requirements

### Ansible Controller / AWX Execution Environment

The execution environment requires:

- Python compatible with the installed Ansible release.
- `ansible-core`.
- `ansible.netcommon`.
- `cisco.ios`.
- Git.
- An SSH client.
- Network reachability to managed devices and the Git server.
- Appropriate permissions for local backup storage.

Install the required collections:

```bash
ansible-galaxy collection install ansible.netcommon cisco.ios
```

Check installed collection versions:

```bash
ansible-galaxy collection list
```

Verify Git availability:

```bash
git --version
```

For production AWX deployments, package the dependencies into a version-controlled execution environment image rather than relying on manually installed collections.

### Network Devices

Managed Cisco devices must:

- Support the selected Ansible network OS and module.
- Permit the required CLI access.
- Have working management-plane connectivity from the execution environment.
- Use appropriate authentication and authorization privileges.
- Permit retrieval of the required running configuration.

### Git Server

The Git server must provide:

- An accessible repository.
- An SSH key authorized to write to the repository.
- A valid and trusted SSH host key.
- Permission to push to the `main` branch, subject to the repository's branch protection rules.

## AWX Integration

The playbooks can be executed through AWX job templates.

Recommended configuration:

1. Create an inventory containing the managed devices.
2. Define a `cisco` inventory group for Cisco devices.
3. Configure device authentication through AWX credentials.
4. Configure the Git SSH credential for the execution environment.
5. Ensure required Ansible collections are available.
6. Configure the project to use the appropriate repository and branch.
7. Create a job template referencing the backup playbook.
8. Validate the workflow manually before enabling scheduled execution.

### Suggested Execution Policies

- Use dedicated credentials with the minimum privileges required.
- Limit concurrent jobs that push to the same Git branch.
- Configure job timeouts appropriate to the number of devices.
- Monitor failed device connections and Git push failures.
- Retain AWX job output for operational troubleshooting.
- Verify that local backup storage is persistent.

## Security Considerations

Configuration backups may contain sensitive information, including network addressing, routing policies, access-control rules, SNMP configuration, and authentication-related settings.

Apply the following controls:

- Never commit passwords, private keys, API tokens, or other secrets.
- Restrict access to backup repositories.
- Protect the `main` branch according to operational requirements.
- Use SSH host-key verification rather than disabling host-key checking.
- Store credentials in AWX credential objects or an approved secrets-management system.
- Avoid logging complete configurations or sensitive task output.
- Restrict permissions on local backup files and directories.
- Define backup retention and access policies.
- Review configuration diffs before applying a restoration.

A Git repository containing network configurations should be treated as a sensitive operational asset.

## Operational Reliability

### Partial Device Failures

A failed device backup must not overwrite its last known-good configuration in Git.

The workflow should process successful device backups without treating an unsuccessful collection as an empty configuration.

### Concurrent Executions

Multiple AWX jobs may attempt to push to `main` concurrently.

For production deployments, serialize the Git commit/push stage where practical. If concurrent execution is required, implement bounded retries with fetch/rebase handling and explicit failure reporting.

Concurrent changes to the same device file may require manual conflict resolution.

### Backup Retention

Timestamped local backups can consume significant storage over time.

Define a retention policy based on operational and compliance requirements. Ensure that local retention and Git history retention are managed independently.

### Validation and Testing

Before production deployment, validate the following scenarios:

- Initial backup of a device.
- Repeated backup with no configuration changes.
- Backup following a real configuration change.
- Configuration containing supported platform-specific metadata.
- Device connectivity failure.
- Invalid or empty backup output.
- Git authentication failure.
- Git push rejection caused by concurrent changes.
- Recovery of a previous configuration from Git history.

## Troubleshooting

### Cisco Connection Failure

Check:

- Management IP reachability.
- SSH access and device credentials.
- AWX execution environment network connectivity.
- Ansible collection compatibility.
- Device CLI privilege level.

### Git Clone Failure

Check:

- Repository URL.
- GitLab SSH host-key trust.
- SSH key availability inside the execution environment.
- Repository permissions.
- Network connectivity to the Git server.

Test repository access:

```bash
git ls-remote git@gitlab.example.local:ip-team/awx-ansible.git
```

### No Git Commit Created

No commit is expected when normalized configuration content is identical to the tracked version.

If a real configuration change is missing from Git, inspect the collected backup, normalization rules, AWX task results, and Git staging output.

### Local Backup Missing

Verify that:

- The backup directory exists.
- The execution environment can write to the configured path.
- The path is mounted to persistent storage where necessary.
- The backup task completed successfully.
- AWX is not deleting temporary or container-local storage.

## Contributing

When adding a new playbook:

1. Document its purpose and supported platforms.
2. Define the required inventory variables and credentials.
3. Validate inputs and task results.
4. Avoid embedding secrets in source files.
5. Use Ansible modules where appropriate instead of unnecessary shell commands.
6. Include failure handling and meaningful task output.
7. Test successful, unchanged, and failed execution paths.
8. Document any operational dependencies.

Keep automation modular and reusable. Changes to production workflows should be reviewed and tested before deployment.

## License

Specify the applicable license before distributing this repository publicly or allowing external reuse.

---

**Network Automation | Ansible | AWX | Cisco IOS | Git-Based Configuration Management**
```
