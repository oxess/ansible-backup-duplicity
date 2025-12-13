# Ansible Role: Duplicity Backup

Ansible role for managing [Duplicity](http://duplicity.nongnu.org/) backups with support for multiple storage backends, flexible scheduling, GPG encryption, and optional Zabbix monitoring with auto-detection.

## Features

- **Multiple Backup Destinations**: S3, SFTP, Local filesystem
- **Flexible Encryption**: GPG key-based or symmetric passphrase encryption
- **Configurable Scheduling**: Cron-based with preset options (hourly, daily, weekly, monthly)
- **Retention Policies**: Time-based cleanup and chain-based rotation
- **Zabbix Auto-Detection**: Automatically configures monitoring when Zabbix agent is present
- **Restore Utilities**: Comprehensive scripts for full, partial, and point-in-time recovery
- **Multi-Distribution Support**: Debian 11/12, Ubuntu 20.04/22.04/24.04

## Requirements

- Ansible 2.12 or higher
- Supported operating systems:
  - Debian 11 (Bullseye), 12 (Bookworm)
  - Ubuntu 20.04 (Focal), 22.04 (Jammy), 24.04 (Noble)

## Installation

### From Ansible Galaxy

```bash
ansible-galaxy install oxess.duplicity
```

### From GitHub

```bash
ansible-galaxy install git+https://github.com/oxess/ansible-backup-duplicity.git,main
```

## Quick Start

### Local Backup

```yaml
- hosts: servers
  roles:
    - role: oxess.duplicity
      vars:
        duplicity_destination: "file:///mnt/backup"
        duplicity_passphrase: "{{ vault_backup_passphrase }}"
        duplicity_include_paths:
          - /etc
          - /var/www
          - /home
```

### S3 Backup

```yaml
- hosts: servers
  roles:
    - role: oxess.duplicity
      vars:
        duplicity_destination: "s3://s3.amazonaws.com/my-bucket/backups"
        duplicity_passphrase: "{{ vault_backup_passphrase }}"
        duplicity_aws_access_key_id: "{{ vault_aws_key }}"
        duplicity_aws_secret_access_key: "{{ vault_aws_secret }}"
        duplicity_include_paths:
          - /etc
          - /var/www
```

### SFTP Backup

```yaml
- hosts: servers
  roles:
    - role: oxess.duplicity
      vars:
        duplicity_destination: "sftp://backup-user@backup.example.com/backups"
        duplicity_passphrase: "{{ vault_backup_passphrase }}"
        duplicity_include_paths:
          - /etc
          - /var/www
```

## Role Variables

### Required Variables

| Variable                | Description                                      |
|-------------------------|--------------------------------------------------|
| `duplicity_destination` | Backup destination URL (s3://, sftp://, file://) |
| `duplicity_passphrase`  | Encryption passphrase (use Ansible Vault)        |

### Backup Sources

| Variable                     | Default             | Description              |
|------------------------------|---------------------|--------------------------|
| `duplicity_include_paths`    | `[/etc, /var/www]`  | Directories to backup    |
| `duplicity_exclude_patterns` | `["**/*.log", ...]` | Glob patterns to exclude |

### Encryption

| Variable                      | Default     | Description                          |
|-------------------------------|-------------|--------------------------------------|
| `duplicity_encryption_method` | `symmetric` | `gpg`, `symmetric`, or `none`        |
| `duplicity_passphrase`        | `""`        | Encryption passphrase                |
| `duplicity_gpg_key`           | `""`        | GPG key ID for asymmetric encryption |
| `duplicity_sign_key`          | `""`        | GPG key for signing backups          |

### Backend Credentials

| Variable                          | Default | Description                      |
|-----------------------------------|---------|----------------------------------|
| `duplicity_aws_access_key_id`     | `""`    | AWS/S3 access key                |
| `duplicity_aws_secret_access_key` | `""`    | AWS/S3 secret key                |
| `duplicity_s3_endpoint_url`       | `""`    | Custom S3 endpoint (MinIO, etc.) |
| `duplicity_sftp_password`         | `""`    | SFTP password (prefer SSH keys)  |

### Schedule

| Variable                  | Default | Description                                    |
|---------------------------|---------|------------------------------------------------|
| `duplicity_cron_enabled`  | `true`  | Enable cron job                                |
| `duplicity_cron_hour`     | `"3"`   | Hour to run backup                             |
| `duplicity_cron_minute`   | `"0"`   | Minute to run backup                           |
| `duplicity_cron_schedule` | `""`    | Preset: `hourly`, `daily`, `weekly`, `monthly` |

### Retention

| Variable                       | Default | Description                            |
|--------------------------------|---------|----------------------------------------|
| `duplicity_remove_older_than`  | `"4W"`  | Remove backups older than (1D, 1W, 1M) |
| `duplicity_full_if_older_than` | `"1W"`  | Force full backup after this period    |
| `duplicity_keep_full_chains`   | `2`     | Number of full chains to keep          |

### Zabbix Integration

| Variable                          | Default                            | Description                          |
|-----------------------------------|------------------------------------|--------------------------------------|
| `duplicity_zabbix_enabled`        | `"auto"`                           | `auto`, `true`, or `false`           |
| `duplicity_zabbix_agent_conf_dir` | `/etc/zabbix/zabbix_agentd.conf.d` | Zabbix agent config directory        |
| `duplicity_zabbix_check_hours`    | `24`                               | Hours to check for successful backup |

### Other Options

| Variable                  | Default | Description                   |
|---------------------------|---------|-------------------------------|
| `duplicity_volsize`       | `200`   | Volume size in MB             |
| `duplicity_max_retries`   | `3`     | Max retry attempts on failure |
| `duplicity_log_enabled`   | `true`  | Enable logging to file        |
| `duplicity_log_to_syslog` | `true`  | Log to syslog                 |

## Installed Scripts

The role installs several utility scripts:

| Script              | Description                           |
|---------------------|---------------------------------------|
| `duplicity-backup`  | Main backup script (run by cron)      |
| `duplicity-status`  | Show backup collection status         |
| `duplicity-restore` | Restore utility with multiple options |
| `duplicity-list`    | List files in backup archive          |

## Backup & Restore Operations

### Manual Backup

```bash
sudo duplicity-backup
```

### Check Backup Status

```bash
duplicity-status
```

### List Available Backups

```bash
duplicity-restore list
```

### Full Restore

```bash
# Restore to /restore directory
duplicity-restore restore

# Restore to custom directory
duplicity-restore restore / /tmp/full-restore
```

### Restore Specific Directory

```bash
# Restore /etc/nginx to /tmp/nginx-restore
duplicity-restore restore /etc/nginx /tmp/nginx-restore
```

### Point-in-Time Restore

```bash
# Restore from 3 days ago
duplicity-restore restore --time 3D /etc /tmp/etc-restore

# Restore from specific date
duplicity-restore restore --time 2024-01-15 /var/www /tmp/www-restore
```

### Restore Single File

```bash
duplicity-restore file /etc/nginx/nginx.conf /tmp/nginx.conf.bak
```

### Verify Backup Integrity

```bash
duplicity-restore verify
```

### Dry-Run (Preview)

```bash
duplicity-restore restore --dry-run /etc
```

### List Files in Backup

```bash
# List all files
duplicity-list

# List files under /etc
duplicity-list /etc

# List files from 3 days ago
duplicity-list --time 3D /var/www
```

## Disaster Recovery

1. **Install duplicity** on new server:
   ```bash
   apt install duplicity python3-boto3 python3-paramiko gnupg
   ```

2. **Copy credentials** (GPG keys or set environment variables):
   ```bash
   export PASSPHRASE="your-backup-passphrase"
   export AWS_ACCESS_KEY_ID="your-key"
   export AWS_SECRET_ACCESS_KEY="your-secret"
   ```

3. **List available backups**:
   ```bash
   duplicity collection-status s3://bucket/path
   ```

4. **Restore**:
   ```bash
   duplicity restore s3://bucket/path /restore
   ```

5. **Verify** restored files and move to final locations.

## Zabbix Monitoring

When Zabbix agent is detected (or explicitly enabled), the role configures monitoring with these UserParameters:

| Key                                 | Description                         |
|-------------------------------------|-------------------------------------|
| `duplicity.backup.success_count`    | Successful backups in last N hours  |
| `duplicity.backup.running`          | Is backup currently running (1/0)   |
| `duplicity.backup.last_success`     | Timestamp of last successful backup |
| `duplicity.backup.error_count`      | Errors in last 24 hours             |
| `duplicity.backup.collection_count` | Number of backup sets               |

## Security Considerations

- **Credentials**: Store in Ansible Vault, deployed with mode 0600
- **Scripts**: Owned by root, mode 0750
- **Config Directory**: Mode 0750
- **Lock File**: Prevents concurrent backup runs
- **GPG Keys**: Managed externally, referenced by ID

## Testing

Run Molecule tests:

```bash
pip install molecule molecule-plugins[docker]
molecule test
```

## License

MIT

## Author

oxess
