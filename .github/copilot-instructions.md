# AWS to Azure Data Migration - AI Coding Agent Instructions

## Project Overview
This repository automates cloud data migration from AWS to Azure using Hitachi storage systems. It orchestrates storage volume creation, journal configuration, and hybrid data replication (HUR) pairing across cloud environments.

## Architecture & Key Components

### Main Workflow (Sequential Playbooks)
- **`1_create_aws_cloud_journal.yml`**: Creates AWS journal volume via `hsds` CLI commands to Hitachi storage
- **`2_create_azure_cloud_svol.yml`**: Creates Azure secondary volume and establishes server connections
- **`3_create_azure_cloud_journal.yml`**: Creates Azure journal volume (mirrors AWS journal creation pattern)
- **`4_create_aws_cloud_hur_pair.yml`**: Establishes HUR replication pair between AWS/Azure; configures HORCM (Hitachi Remote Replication Manager)

### Execution Environment (`cloud_ee/`)
Custom Ansible Execution Environment (EE) built via `execution_environment.yml`:
- **Base image**: CentOS Stream 9
- **Key packages**: `hsds-cli` (Hitachi storage CLI), `HORCM` (replication manager), `ansible-core`, `ansible-runner`
- **Build process**: Dockerfile-style multi-stage build copying configs and HORCM templates

### Configuration & Secrets
- **`ansible_vault_storage_var.yml`**: Centralized vault containing storage credentials and infrastructure IDs (IP addresses, serial numbers, controller IDs)
  - Expected to be encrypted with `ansible-vault` before deployment
  - Variables follow naming convention: `{aws|az}_cloud_*` for environment separation

## Critical Patterns & Conventions

### 1. **Shell Command Parsing Pattern**
All playbooks extract remote replication parameters via chained shell pipes:
```yaml
hsds --ignore_certificate_errors --host "{{ address }}" ... | grep "^Pattern" | tr -d " " | cut -d":" -f2
```
- Always include `set -o pipefail` to fail on grep mismatches
- Parse output by anchoring to expected line prefixes (e.g., `"^Job"`, `"^Volume"`, `"^Control"`)
- Strip whitespace with `tr -d " "` and extract fields with `cut -d":" -f2`

### 2. **Retry Pattern with State Files**
Tasks use file existence checks to track completion:
```yaml
- ansible.builtin.stat: path: /tmp/journal_created
- when: not journal_created.stat.exists
  args:
    creates: /tmp/journal_created
```
This idiomatically handles idempotency across container restarts.

### 3. **Multi-Step Data Flow**
Jobs are async; workflows always:
1. Create job → capture Job ID
2. Poll job status with `job_list` (retries: 10, delay: 30s)
3. Extract final resource ID from job output
4. Use ID for downstream operations

### 4. **Jinja2 Template Injection**
HORCM config files use `*.j2` templates populated during playbook execution:
- `horcm100.conf.j2`, `horcm101.conf.j2` pre-exist in `cloud_ee/`; dynamically filled with storage node IPs and path group IDs
- Templates injected into running container via `ansible.builtin.template` task

### 5. **Storage Variable Namespace**
- **AWS-side**: `aws_cloud_*`, `aws_storage_*`, `aws_udp_port`, `pvol_ldev_id` (primary volume)
- **Azure-side**: `az_cloud_*`, `az_storage_*`, `az_udp_port`, `svol_ldev_id` (secondary volume)
- Note: Journal numbers (e.g., `journal_number: 100`) link to HORCM instances (`HORCMINST=100`)

## Developer Workflows

### Building the Execution Environment
```bash
ansible-builder build --container-runtime=podman -c cloud_ee/context/ -t aws2azure-ee:latest
```
Output: OCI-compliant container image with hsds-cli, HORCM, and ansible pre-installed.

### Running Playbooks
1. **Prepare vault**:
   ```bash
   ansible-vault encrypt ansible_vault_storage_var.yml  # First time
   ```
2. **Execute workflow**:
   ```bash
   ansible-playbook 1_create_aws_cloud_journal.yml \
     --ask-vault-pass -i localhost,
   ```
3. **Sequentially run** `1_*`, `2_*`, `3_*`, `4_*` in order (data dependencies).

### Testing
- Playbooks include `debug:` tasks that print intermediate values (job IDs, volume IDs)
- Verify output by checking status commands in each playbook
- Failed steps leave state files (e.g., `/tmp/journal_created`) unconsumed, allowing re-runs

## Integration Points

### External APIs
- **Hitachi VSP One Block (via `hsds`)**: Core API for volume/journal management
  - Endpoint: `{{ storage_address }}` (IP from vault)
  - Commands: `volume_create`, `job_list`, `journal_create`, `storage_controller_show`, `remotepath_group_list`
- **HORCM CLI (`/HORCM/usr/bin/`)**: Direct replication management
  - Commands: `raidcom`, `paircreate`, `pairevtwait`, `pairsplit`

### Key Dependencies
- **Collections**: `hitachivantara.vspone_block` (vsphere-level APIs), `vmware.vmware`, `community.vmware`
- **Python**: `pyVmomi>=8.0.3.0.1`, `aiohttp` (from `requirements.txt`)
- **System**: `python3`, `tar`, `gzip` (from `bindep.txt`)

## Common Modifications

### Adding New Storage Configuration
1. Update `ansible_vault_storage_var.yml` with new `{aws|az}_cloud_*` variables
2. Reference in playbook via `{{ variable_name }}`
3. Ensure naming matches environment (aws/az prefix)

### Extending HORCM Setup
1. Edit `horcm100.conf.j2` / `horcm101.conf.j2` templates
2. Add new Jinja2 variables from playbook facts (e.g., `{{ new_storage_ip }}`)
3. Rebuild EE image to include updated templates

### Adding Validation Steps
Use `shell` task with `until` + `retries` for polling:
```yaml
until: result.rc == 0
retries: 10
delay: 30
```
