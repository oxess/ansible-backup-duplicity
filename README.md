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
  - Debian 12 (Bookworm), 13 (Trixie)
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

## AWS S3 Permissions

When using S3 as a backup destination, Duplicity requires specific permissions to operate. Below is the minimal IAM policy needed:

### Minimal IAM Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::your-bucket-name"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::your-bucket-name/your-backup-path/*"
    }
  ]
}
```

### Required Permissions Explained

- **`s3:ListBucket`**: Required to list existing backup sets and verify backup chains
- **`s3:GetBucketLocation`**: Needed to determine the bucket's region for proper API calls
- **`s3:GetObject`**: Required to download backup archives during restore operations and to verify existing backups
- **`s3:PutObject`**: Required to upload new backup archives
- **`s3:DeleteObject`**: Required for cleanup operations when removing old backups according to retention policies

### IAM User Setup Example

1. Create an IAM user specifically for backups:
   ```bash
   aws iam create-user --user-name duplicity-backup
   ```

2. Create and attach the policy:
   ```bash
   aws iam create-policy --policy-name DuplicityBackupPolicy --policy-document file://duplicity-policy.json
   aws iam attach-user-policy --user-name duplicity-backup --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/DuplicityBackupPolicy
   ```

3. Create access keys:
   ```bash
   aws iam create-access-key --user-name duplicity-backup
   ```

### S3-Compatible Storage (MinIO, Wasabi, etc.)

For S3-compatible storage providers, use the same permissions structure but with the appropriate endpoint:

```yaml
- hosts: servers
  roles:
    - role: oxess.duplicity
      vars:
        duplicity_destination: "s3://your-minio-server/bucket/backups"
        duplicity_s3_endpoint_url: "https://your-minio-server:9000"
        duplicity_aws_access_key_id: "{{ vault_minio_access_key }}"
        duplicity_aws_secret_access_key: "{{ vault_minio_secret_key }}"
```

## Using Ansible Vault for Sensitive Data

Ansible Vault should be used to protect sensitive information like passwords, API keys, and passphrases. Here's a step-by-step guide:

### Step 1: Create a Vault File

Create an encrypted vault file to store your sensitive variables:

```bash
ansible-vault create group_vars/all/vault.yml
```

You'll be prompted to create a vault password. Remember this password as you'll need it to decrypt the file.

### Step 2: Add Sensitive Variables

Edit the vault file and add your sensitive data:

```bash
ansible-vault edit group_vars/all/vault.yml
```

Add the following variables (example for AWS S3):

```yaml
# Encryption passphrase
vault_backup_passphrase: "your-strong-backup-passphrase-here"

# AWS Credentials
vault_aws_access_key_id: "AKIAIOSFODNN7EXAMPLE"
vault_aws_secret_access_key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"

# For SFTP (if needed)
vault_sftp_password: "your-sftp-password"

# For GPG encryption (if using)
vault_gpg_passphrase: "your-gpg-key-passphrase"
```

### Step 3: Create Your Playbook

Create a playbook that references the vault variables:

```yaml
---
# playbook.yml
- hosts: backup_servers
  become: yes
  roles:
    - role: oxess.duplicity
      vars:
        # S3 destination using vault variables
        duplicity_destination: "s3://s3.amazonaws.com/your-bucket-name/backups/{{ inventory_hostname }}"

        # Use vault-encrypted variables
        duplicity_passphrase: "{{ vault_backup_passphrase }}"
        duplicity_aws_access_key_id: "{{ vault_aws_access_key_id }}"
        duplicity_aws_secret_access_key: "{{ vault_aws_secret_access_key }}"

        # Backup configuration
        duplicity_include_paths:
          - /etc
          - /var/www
          - /home

        # Schedule daily backups at 2 AM
        duplicity_cron_schedule: "daily"
        duplicity_cron_hour: "2"
        duplicity_cron_minute: "0"

        # Retention: keep 4 weeks of backups
        duplicity_remove_older_than: "4W"
        duplicity_full_if_older_than: "1W"
```

### Step 4: Run the Playbook

Execute the playbook with vault password:

```bash
# Option 1: Enter vault password interactively
ansible-playbook -i inventory playbook.yml --ask-vault-pass

# Option 2: Use a vault password file
echo "your-vault-password" > .vault_pass
chmod 600 .vault_pass
ansible-playbook -i inventory playbook.yml --vault-password-file .vault_pass

# Option 3: Set environment variable (for CI/CD)
export ANSIBLE_VAULT_PASSWORD_FILE=.vault_pass
ansible-playbook -i inventory playbook.yml
```

### Complete AWS S3 Example with Vault

1. **Create vault with AWS credentials:**

```bash
ansible-vault create group_vars/all/vault.yml
```

Content:
```yaml
vault_backup_passphrase: "MySecureBackup#Pass2024!"
vault_aws_access_key_id: "AKIAIOSFODNN7EXAMPLE"
vault_aws_secret_access_key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
vault_aws_bucket: "your-bucket-name"
```

2. **Create the playbook (`backup-setup.yml`):**

```yaml
---
- name: Configure Duplicity backups to AWS S3
  hosts: production
  become: yes
  vars:
    backup_bucket: "{{ vault_aws_bucket }}"
    backup_prefix: "{{ ansible_hostname }}/{{ ansible_date_time.year }}"

  roles:
    - role: oxess.duplicity
      vars:
        # S3 destination with dynamic path
        duplicity_destination: "s3://s3.amazonaws.com/{{ backup_bucket }}/{{ backup_prefix }}"

        # Encryption
        duplicity_encryption_method: "symmetric"
        duplicity_passphrase: "{{ vault_backup_passphrase }}"

        # AWS credentials from vault
        duplicity_aws_access_key_id: "{{ vault_aws_access_key_id }}"
        duplicity_aws_secret_access_key: "{{ vault_aws_secret_access_key }}"

        # What to backup
        duplicity_include_paths:
          - /etc
          - /var/www
          - /home
          - /var/lib/mysql

        duplicity_exclude_patterns:
          - "**/cache/**"
          - "**/*.log"
          - "**/tmp/**"
          - "**/.git/**"

        # Schedule: Daily at 3 AM
        duplicity_cron_enabled: true
        duplicity_cron_schedule: "daily"
        duplicity_cron_hour: "3"
        duplicity_cron_minute: "0"

        # Retention policy
        duplicity_remove_older_than: "30D"
        duplicity_full_if_older_than: "7D"
        duplicity_keep_full_chains: 3

        # Performance
        duplicity_volsize: 500

        # Monitoring
        duplicity_zabbix_enabled: "auto"
        duplicity_log_enabled: true
```

3. **Deploy the configuration:**

```bash
# Test run (check mode)
ansible-playbook -i inventory backup-setup.yml --ask-vault-pass --check

# Actual deployment
ansible-playbook -i inventory backup-setup.yml --ask-vault-pass

# Verify the backup is working
ansible all -i inventory -m command -a "sudo duplicity-status" --ask-vault-pass
```

### Vault Best Practices

1. **Never commit unencrypted vault files** - Add to `.gitignore`:
   ```
   *.vault
   .vault_pass
   vault_password.txt
   ```

2. **Use separate vaults for different environments:**
   ```
   group_vars/production/vault.yml
   group_vars/staging/vault.yml
   group_vars/development/vault.yml
   ```

3. **Rotate credentials regularly** - Re-encrypt vault files:
   ```bash
   ansible-vault rekey group_vars/all/vault.yml
   ```

4. **View vault contents without editing:**
   ```bash
   ansible-vault view group_vars/all/vault.yml
   ```

5. **Encrypt existing files:**
   ```bash
   ansible-vault encrypt existing_vars.yml
   ```

6. **Decrypt temporarily for editing:**
   ```bash
   ansible-vault decrypt vault.yml
   # Edit the file
   ansible-vault encrypt vault.yml
   ```

### CI/CD Integration

For automated deployments, store the vault password securely:

**GitHub Actions example:**
```yaml
- name: Run Ansible Playbook
  env:
    ANSIBLE_VAULT_PASSWORD: ${{ secrets.ANSIBLE_VAULT_PASSWORD }}
  run: |
    echo "$ANSIBLE_VAULT_PASSWORD" > .vault_pass
    ansible-playbook -i inventory playbook.yml --vault-password-file .vault_pass
    rm .vault_pass
```

**GitLab CI example:**
```yaml
deploy:
  script:
    - echo "$ANSIBLE_VAULT_PASSWORD" > .vault_pass
    - ansible-playbook -i inventory playbook.yml --vault-password-file .vault_pass
    - rm .vault_pass
  variables:
    ANSIBLE_VAULT_PASSWORD: $CI_ANSIBLE_VAULT_PASSWORD
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
