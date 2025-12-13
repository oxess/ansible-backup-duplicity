# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2024-12-13

### Added

- Initial release of the Ansible Duplicity Backup role
- Support for multiple backup destinations:
  - Amazon S3 and S3-compatible storage
  - SFTP/SSH
  - Local filesystem
- Encryption options:
  - GPG key-based encryption
  - Symmetric passphrase encryption
  - Option to disable encryption
- Flexible scheduling with cron:
  - Custom cron expressions
  - Preset schedules (hourly, daily, weekly, monthly)
- Backup retention policies:
  - Time-based cleanup (`remove-older-than`)
  - Chain-based retention (`keep_full_chains`)
- Comprehensive restore utilities:
  - Full restore
  - Partial/directory restore
  - Single file restore
  - Point-in-time recovery
  - Dry-run preview mode
- Zabbix monitoring integration:
  - Auto-detection of Zabbix agent
  - Custom UserParameters for backup monitoring
  - Success/failure tracking
- Additional features:
  - Retry logic for failed backups
  - Lock file to prevent concurrent runs
  - Logging to file and syslog
  - Logrotate configuration
- Multi-distribution support:
  - Debian 11 (Bullseye)
  - Debian 12 (Bookworm)
  - Ubuntu 20.04 (Focal)
  - Ubuntu 22.04 (Jammy)
  - Ubuntu 24.04 (Noble)
- Molecule testing with Docker
- Galaxy-ready metadata
- Comprehensive documentation

### Security

- Credentials stored in separate file with mode 0600
- Scripts owned by root with mode 0750
- Support for Ansible Vault integration
- GPG encryption support for secure backups
