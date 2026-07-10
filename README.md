# Backup-System
A fully automated backup system using Azure Blob Storage with versioning, lifecycle management policies that control cost by moving older data to cheaper storage tiers, and a Logic Apps workflow that sends a daily backup confirmation email.

# Project 2: Automated Backup System



## The Business Problem You Are Solving

For most small businesses, the backup strategy is someone occasionally copying files to an external drive. When that person is on holiday, busy, or just forgets — nothing gets backed up. When something goes wrong, and something always eventually goes wrong, the data is simply gone.

This project replaces that manual, unreliable process with a system that:

- Automatically replicates every file across multiple Azure data centers the moment it is uploaded
- Keeps every previous version of every file, so anything accidentally deleted or overwritten can be restored
- Automatically moves older files to cheaper storage tiers after 30 days and archives them after 90 days — keeping costs under control without manual intervention
- Sends a confirmation email every morning so the business owner knows the backup system is working without having to check

The business outcome is simple and powerful: no more lost data, no more manual processes, no more hoping someone remembered.



## What Gets Built

```
rg-backup-[yourname]
├── Storage Account (primary backup store)
│   ├── Container: documents
│   ├── Container: database-exports
│   ├── Container: application-files
│   ├── Blob Versioning (enabled)
│   └── Lifecycle Policy (30d cool → 90d archive)
├── Log Analytics Workspace
├── Storage Diagnostic Settings     → logs to Log Analytics
├── Logic App Workflow              → daily backup confirmation email
└── Monitor Alert Rule              → fires if storage writes stop
```



## Prerequisites

If you completed Project 1 or the data lab, Terraform and the Azure CLI are already installed. Skip to Folder Setup.

### Mac

```bash
# Install Homebrew if needed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Terraform
brew tap hashicorp/tap && brew install hashicorp/tap/terraform
terraform --version

# Install Azure CLI
brew install azure-cli
az login
az account set --subscription "Azure subscription 1"
```

### Windows (PowerShell)

Download Terraform from https://developer.hashicorp.com/terraform/install and Azure CLI from https://aka.ms/installazurecliwindows, then:

```powershell
[Environment]::SetEnvironmentVariable("PATH", $env:PATH + ";C:\terraform", "User")
az login
az account set --subscription "Azure subscription 1"
```



## Folder Setup

**Mac:**

```bash
mkdir ~/backup-system-001
cd ~/backup-system-001
touch main.tf variables.tf outputs.tf terraform.tfvars
```

**Windows (PowerShell):**

```powershell
New-Item -ItemType Directory -Path "$HOME\backup-system-001"
cd "$HOME\backup-system-001"
New-Item -ItemType File main.tf, variables.tf, outputs.tf, terraform.tfvars
```



## Step 1 — Write variables.tf

```hcl
variable "yourname" {
  description = "Your name, lowercase, no spaces. Used to make resource names unique."
  type        = string
}

variable "location" {
  type    = string
  default = "East US"
}

variable "alert_email" {
  description = "Email address to receive daily backup confirmation."
  type        = string
}

variable "tags" {
  type = map(string)
  default = {
    project     = "backup-system"
    environment = "dev"
    managed_by  = "terraform"
  }
}
```



## Step 2 — Write terraform.tfvars

```hcl
yourname    = "nafesse"
location    = "East US"
alert_email = "your.email@example.com"
```



## Step 3 — Write main.tf

### Provider

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

data "azurerm_client_config" "current" {}
```

### Resource group

```hcl
resource "azurerm_resource_group" "main" {
  name     = "rg-backup-${var.yourname}"
  location = var.location
  tags     = var.tags
}
```

### Storage account

This is the core of the backup system. Every file uploaded here is automatically replicated across multiple physical Azure data centers — not just different rooms in the same building, but geographically separate facilities.

`account_replication_type = "GRS"` stands for Geo-Redundant Storage. Azure keeps your data in your primary region (East US) and asynchronously copies it to a secondary region (West US) automatically. If an entire region goes offline, your data still exists in the secondary. For a backup system, GRS is the correct choice — LRS (used in the data lab) only replicates within a single region.

`min_tls_version = "TLS1_2"` enforces that all connections to the storage account use TLS 1.2 or higher. Older TLS versions have known vulnerabilities and should not be used for backup data.

`blob_properties` with `versioning_enabled = true` is what makes this a real backup system rather than just a file store. Every time a file is overwritten or deleted, Azure keeps the previous version. You can restore any file to any point in time. A user who accidentally deletes a folder or saves a corrupted file over a good one does not lose data permanently.

`delete_retention_policy` with `days = 30` means that even after a blob is deleted, Azure retains it in a soft-deleted state for 30 days before permanently removing it. This is a safety net on top of versioning.

```hcl
resource "azurerm_storage_account" "backup" {
  name                     = "stbackup${var.yourname}"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "GRS"
  min_tls_version          = "TLS1_2"

  blob_properties {
    versioning_enabled = true

    delete_retention_policy {
      days = 30
    }

    container_delete_retention_policy {
      days = 30
    }
  }

  tags = var.tags
}
```

### Storage containers

Containers are the top-level organizational unit inside a storage account — similar to folders, but at the root level. Separating data by type (documents, database exports, application files) matters enormously when you need to restore something under pressure. You do not want to be searching through a mixed pile of files during an incident.

`container_access_type = "private"` means no public internet access. Files can only be accessed by authenticated Azure identities or connection strings. Backup data should never be publicly readable.

```hcl
resource "azurerm_storage_container" "documents" {
  name                  = "documents"
  storage_account_name  = azurerm_storage_account.backup.name
  container_access_type = "private"
}

resource "azurerm_storage_container" "database_exports" {
  name                  = "database-exports"
  storage_account_name  = azurerm_storage_account.backup.name
  container_access_type = "private"
}

resource "azurerm_storage_container" "application_files" {
  name                  = "application-files"
  storage_account_name  = azurerm_storage_account.backup.name
  container_access_type = "private"
}
```

### Lifecycle management policy

This is what keeps backup costs from growing unbounded over time. The policy defines rules that automatically move files between storage tiers based on how old they are.

Azure has four storage tiers — Hot, Cool, Cold, and Archive — each progressively cheaper to store but more expensive to read. Hot is for data accessed frequently. Cool is for data accessed occasionally. Archive is for data that almost never needs to be read but must be retained for compliance or recovery.

The `base_blob` rule applies to the current (live) version of each file. After 30 days of not being modified, a file moves from Hot to Cool storage automatically. After 90 days, it moves to Archive. After 365 days, the file is deleted.

The `version` rule applies to older versions that have been superseded. These are kept for 30 days (long enough to catch any data corruption or accidental deletion) then permanently removed. Keeping old versions indefinitely would grow storage costs without limit.

`filters` with `prefix_match = ["documents/", "database-exports/", "application-files/"]` means this policy applies across all three containers.

```hcl
resource "azurerm_storage_management_policy" "lifecycle" {
  storage_account_id = azurerm_storage_account.backup.id

  rule {
    name    = "backup-lifecycle"
    enabled = true

    filters {
      blob_types   = ["blockBlob"]
      prefix_match = ["documents/", "database-exports/", "application-files/"]
    }

    actions {
      base_blob {
        tier_to_cool_after_days_since_modification_greater_than    = 30
        tier_to_archive_after_days_since_modification_greater_than = 90
        delete_after_days_since_modification_greater_than          = 365
      }

      version {
        delete_after_days_since_creation = 30
      }
    }
  }
}
```

### Log Analytics Workspace

```hcl
resource "azurerm_log_analytics_workspace" "main" {
  name                = "law-backup-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
  tags                = var.tags
}
```

### Storage diagnostic settings

This routes storage account logs and metrics into Log Analytics. The `StorageWrite` log category records every file write — which is the signal that backups are landing. `StorageRead` and `StorageDelete` record reads and deletions. Together these give you a complete audit trail of everything that has happened to your backup data.

`metric` with category `Transaction` sends metrics about storage operations (request count, latency, errors) to Log Analytics, making it possible to alert on unusual patterns like a sudden drop in write activity.

```hcl
resource "azurerm_monitor_diagnostic_setting" "storage_logs" {
  name                       = "diag-storage-to-law"
  target_resource_id         = "${azurerm_storage_account.backup.id}/blobServices/default"
  log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id

  enabled_log { category = "StorageRead"   }
  enabled_log { category = "StorageWrite"  }
  enabled_log { category = "StorageDelete" }

  metric {
    category = "Transaction"
    enabled  = true
  }
}
```

### Action Group and Logic App for daily confirmation

The Action Group defines where notifications go. The Logic App is what sends the confirmation email each morning. The Logic App is provisioned here and configured in the portal in Step 6.

```hcl
resource "azurerm_monitor_action_group" "backup_alerts" {
  name                = "ag-backup-${var.yourname}"
  resource_group_name = azurerm_resource_group.main.name
  short_name          = "backupalert"

  email_receiver {
    name                    = "owner-email"
    email_address           = var.alert_email
    use_common_alert_schema = true
  }

  tags = var.tags
}

resource "azurerm_logic_app_workflow" "backup_confirmation" {
  name                = "la-backup-confirm-${var.yourname}"
  location            = var.location
  resource_group_name = azurerm_resource_group.main.name
  tags                = var.tags
}
```

### Monitor alert — detect if backups stop

This alert fires if the storage account receives zero write transactions in a 24-hour window. In other words: if nothing is being written to the backup storage, something has gone wrong upstream, and the business owner should know about it.

`metric_name = "Transactions"` is the metric being watched. `operator = "LessThan"` with `threshold = 1` means the alert fires when the transaction count drops below 1 — i.e., zero transactions.

`frequency = "PT1H"` means Azure checks the condition every hour. `window_size = "P1D"` means each check looks at the past 24 hours of data.

`dimension` with `name = "ApiName"` and `values = ["PutBlob", "PutBlock"]` filters the transaction count to only write operations. Without this filter the alert would fire on any quiet period, including normal times when nobody is reading files.

```hcl
resource "azurerm_monitor_metric_alert" "no_writes" {
  name                = "alert-no-backup-writes"
  resource_group_name = azurerm_resource_group.main.name
  scopes              = [azurerm_storage_account.backup.id]
  description         = "Fires if no files have been written to backup storage in 24 hours."
  severity            = 2
  frequency           = "PT1H"
  window_size         = "P1D"

  criteria {
    metric_namespace = "Microsoft.Storage/storageAccounts"
    metric_name      = "Transactions"
    aggregation      = "Total"
    operator         = "LessThan"
    threshold        = 1

    dimension {
      name     = "ApiName"
      operator = "Include"
      values   = ["PutBlob", "PutBlock"]
    }
  }

  action {
    action_group_id = azurerm_monitor_action_group.backup_alerts.id
  }

  tags = var.tags
}
```



## Step 4 — Write outputs.tf

```hcl
output "storage_account_name" {
  value = azurerm_storage_account.backup.name
}

output "storage_account_connection_string" {
  value     = azurerm_storage_account.backup.primary_connection_string
  sensitive = true
}

output "log_analytics_workspace_id" {
  value = azurerm_log_analytics_workspace.main.id
}

output "logic_app_endpoint" {
  value = azurerm_logic_app_workflow.backup_confirmation.access_endpoint
}
```

`sensitive = true` on the connection string means Terraform will not print it in plain text in your terminal. To view it: `terraform output -raw storage_account_connection_string`



## Step 5 — Deploy

**Mac & Windows (PowerShell):**

```bash
terraform init
terraform plan
terraform apply
```

Expect 11 resources to add. Deployment takes 2–3 minutes.



## Step 6 — Configure the Daily Confirmation Logic App

1. In the portal, navigate to `la-backup-confirm-[yourname]`
2. Click **Logic app designer**
3. Click **Add a trigger** → search for **Recurrence** → select **Recurrence**
4. Set **Frequency** to Day and **Interval** to 1. Set **At these hours** to 8 (8:00 AM)
5. Click **+ New step** → search for **Azure Blob Storage** → select **List blobs**
6. Connect to your storage account using the connection string from `terraform output -raw storage_account_connection_string`
7. Set the container to `documents`
8. Click **+ New step** → **Office 365 Outlook** → **Send an email (V2)**
9. Fill in the email:
   - **To:** your alert email
   - **Subject:** `Daily Backup Confirmation — @{formatDateTime(utcNow(), 'yyyy-MM-dd')}`
   - **Body:** `Backup system status: Active. Files in documents container: @{length(body('List_blobs')?['value'])}. All backup containers are protected and healthy.`
10. Click **Save**



## Step 7 — Upload a Test File and Verify Versioning

Upload a test file to confirm the backup system is working.

**Mac:**

```bash
echo "Backup test file created $(date)" > /tmp/backup_test.txt

az storage blob upload \
  --account-name stbackupcharles \
  --container-name documents \
  --name test/backup_test.txt \
  --file /tmp/backup_test.txt \
  --auth-mode login
```

**Windows (PowerShell):**

```powershell
"Backup test file created $(Get-Date)" | Out-File -FilePath "$env:TEMP\backup_test.txt" -Encoding utf8

az storage blob upload `
  --account-name stbackupcharles `
  --container-name documents `
  --name test/backup_test.txt `
  --file "$env:TEMP\backup_test.txt" `
  --auth-mode login
```

Now overwrite the file to create a second version:

**Mac:**

```bash
echo "Updated content — second version $(date)" > /tmp/backup_test.txt

az storage blob upload \
  --account-name stbackupcharles \
  --container-name documents \
  --name test/backup_test.txt \
  --file /tmp/backup_test.txt \
  --auth-mode login \
  --overwrite
```

**Windows (PowerShell):**

```powershell
"Updated content — second version $(Get-Date)" | Out-File -FilePath "$env:TEMP\backup_test.txt" -Encoding utf8

az storage blob upload `
  --account-name stbackupcharles `
  --container-name documents `
  --name test/backup_test.txt `
  --file "$env:TEMP\backup_test.txt" `
  --auth-mode login `
  --overwrite
```

List the versions to confirm both exist:

**Mac:**

```bash
az storage blob list \
  --account-name stbackupcharles \
  --container-name documents \
  --include v \
  --auth-mode login \
  --output table
```

**Windows (PowerShell):**

```powershell
az storage blob list `
  --account-name stbackupcharles `
  --container-name documents `
  --include v `
  --auth-mode login `
  --output table
```

You should see two rows for `test/backup_test.txt` — the current version and one previous version. This confirms versioning is working.



## Verification Checklist

- [ ] Storage account `stbackup[yourname]` exists in the portal
- [ ] Storage account → Data management → Versioning shows Enabled
- [ ] Storage account → Data management → Lifecycle management shows the `backup-lifecycle` rule
- [ ] Three containers exist: `documents`, `database-exports`, `application-files`
- [ ] Logic App `la-backup-confirm-[yourname]` runs on a recurrence trigger
- [ ] Alert rule `alert-no-backup-writes` exists in Monitor → Alerts
- [ ] Test file upload produced two versions in blob list output



## Troubleshooting

| Error | Cause | Resolution |
|---|---|---|
| `StorageErrorCode: BlobAccessTierNotSupported` | Archive tier not available for GRS accounts in some regions | Change `tier_to_archive_after_days` to a longer value or remove the archive rule |
| Logic App blob connection fails | Connection string not entered correctly | Run `terraform output -raw storage_account_connection_string` and paste the full string |
| Alert fires immediately | No write activity at all yet | Upload a file using the test steps above; the alert uses a 24-hour window |
| Two versions not visible in portal | Portal sometimes hides versions by default | In the container view, click **Show versions** toggle |



## Teardown

**Mac:**

```bash
terraform destroy
```

**Windows (PowerShell):**

```powershell
terraform destroy
```
