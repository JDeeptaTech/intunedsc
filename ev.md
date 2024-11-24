``` yaml
- name: Process server backup list
  hosts: localhost
  gather_facts: no
  tasks:
    - name: Define server backup list
      set_fact:
        ev_server_backup_list:
          - "2024-09-04T19:40:22"
          - "2024-09-05T19:39:55"
          - "2024-09-09T11:07:30"
          - "2024-11-23T21:14:00"  # Example latest date
    
    - name: Sort backup dates and get the latest
      set_fact:
        latest_backup_date: "{{ ev_server_backup_list | map('regex_replace', '(T|\\D)', ' ') | map('trim') | map('strptime', '%Y-%m-%d %H:%M:%S') | max }}"
    
    - name: Check if the latest backup is within the last 24 hours
      set_fact:
        backup_within_24h: "{{ (ansible_date_time.iso8601 | to_datetime('%Y-%m-%dT%H:%M:%SZ') - latest_backup_date).total_seconds() <= 86400 }}"
    
    - name: Debug the result
      debug:
        msg: >
          Latest backup date: {{ latest_backup_date.strftime('%Y-%m-%d %H:%M:%S') }}
          Is the latest backup within the last 24 hours? {{ backup_within_24h }}

```

``` powershell
# ev_precheck.ps1

<# 
.SYNOPSIS
    Pre-Upgrade Checks for Veritas Enterprise Vault Upgrade.

.DESCRIPTION
    This script performs pre-upgrade checks, including verifying administrative privileges,
    checking disk space, backing up SQL databases, and stopping EV services.

.PARAMETER SqlServer
    The name or IP address of the SQL Server hosting the EV databases.

.PARAMETER SqlUser
    The SQL login user with backup privileges.

.PARAMETER SqlPassword
    The password for the SQL login user.

.PARAMETER BackupDir
    The directory where SQL backups will be stored.

.PARAMETER EvServices
    An array of Enterprise Vault service display names to be stopped.

.EXAMPLE
    .\ev_precheck.ps1 -SqlServer "sqlserver.example.com" -SqlUser "sa" -SqlPassword "P@ssw0rd" -BackupDir "C:\EVBackups"
#>

param (
    [Parameter(Mandatory=$true)]
    [string]$SqlServer,

    [Parameter(Mandatory=$true)]
    [string]$SqlUser,

    [Parameter(Mandatory=$true)]
    [string]$SqlPassword,

    [Parameter(Mandatory=$true)]
    [string]$BackupDir,

    [Parameter(Mandatory=$false)]
    [string[]]$EvServices = @("Enterprise Vault Admin Service", "Enterprise Vault Directory Service", "Enterprise Vault Storage Service")
)

Write-Host "Starting Pre-Upgrade Checks..."

# Function to check administrative privileges
function Check-AdminPrivileges {
    Write-Host "Checking administrative privileges..."
    If (-Not ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] "Administrator")) {
        Write-Error "You must run this script as an administrator."
        Exit 1
    }
}

# Function to check free disk space
function Check-FreeDiskSpace {
    Write-Host "Checking free disk space on C: drive..."
    $freeSpace = (Get-WMIObject Win32_LogicalDisk -Filter "DeviceID='C:'").FreeSpace
    If ($freeSpace -lt 5GB) {
        Write-Error "Not enough free disk space on C: drive."
        Exit 1
    }
}

# Function to backup SQL databases
function Backup-SqlDatabases {
    Write-Host "Backing up SQL databases..."

    # List of databases to backup
    $databases = @("EVVaultStore", "EVDirectory", "EVMonitoring") # Replace with your actual database names

    # Ensure backup directory exists
    If (-Not (Test-Path -Path $BackupDir)) {
        Write-Host "Creating backup directory at $BackupDir"
        New-Item -Path $BackupDir -ItemType Directory | Out-Null
    }

    # Backup each database
    foreach ($db in $databases) {
        Write-Host "Backing up database: $db"

        $backupFile = Join-Path -Path $BackupDir -ChildPath "$db.bak"

        $backupQuery = "BACKUP DATABASE [$db] TO DISK = N'$backupFile' WITH INIT, STATS = 10"

        try {
            Invoke-Sqlcmd -ServerInstance $SqlServer -Username $SqlUser -Password $SqlPassword -Query $backupQuery -QueryTimeout 0
            Write-Host "Database $db backed up successfully to $backupFile"
        } catch {
            Write-Error "Failed to backup database $db. Error: $_"
            Exit 1
        }
    }
}

# Function to stop Enterprise Vault services
function Stop-EvServices {
    Write-Host "Stopping Enterprise Vault services if they are running..."

    foreach ($serviceName in $EvServices) {
        $service = Get-Service -DisplayName $serviceName -ErrorAction SilentlyContinue
        if ($service -and $service.Status -eq 'Running') {
            Write-Host "Stopping service: $serviceName"
            try {
                Stop-Service -Name $service.Name -Force -ErrorAction Stop
                Write-Host "Service $serviceName stopped successfully."
            } catch {
                Write-Error "Failed to stop service $serviceName. Error: $_"
                Exit 1
            }
        } else {
            Write-Host "Service $serviceName is not running."
        }
    }
}

# Main Execution
try {
    Check-AdminPrivileges
    Check-FreeDiskSpace
    Backup-SqlDatabases
    Stop-EvServices

    Write-Host "Pre-Upgrade Checks Completed Successfully."
    Exit 0
} catch {
    Write-Error "Pre-Upgrade Checks Failed. Error: $_"
    Exit 1
}

```

``` sh
#!/bin/bash

echo "Starting Pre-Upgrade Checks..."

# Check if the script is run as root
if [ "$EUID" -ne 0 ]; then
  echo "You must run this script as root."
  exit 1
fi

# Check available disk space (e.g., require at least 5GB free on /)
free_space=$(df / | tail -1 | awk '{print $4}')
if [ "$free_space" -lt 5242880 ]; then
  echo "Not enough free disk space on the root filesystem."
  exit 1
fi

# Check if Enterprise Vault services are running
ev_services=$(ps aux | grep '[E]nterpriseVault')
if [ -n "$ev_services" ]; then
  echo "Stopping Enterprise Vault services..."
  # Replace with actual commands to stop services
  systemctl stop enterprise_vault.service
fi

echo "Pre-Upgrade Checks Completed Successfully."
exit 0
===================

#!/bin/bash

echo "Starting Enterprise Vault Upgrade..."

INSTALLER_PATH="/tmp/Veritas_Enterprise_Vault_Installer.bin"
RESPONSE_FILE="/tmp/setup.ini"
LOG_FILE="/tmp/ev_upgrade.log"

# Execute the silent upgrade
"$INSTALLER_PATH" -silent -responseFile "$RESPONSE_FILE" > "$LOG_FILE" 2>&1

# Check the installation log for success
if grep -q "Installation Complete" "$LOG_FILE"; then
  echo "Enterprise Vault upgrade completed successfully."
  exit 0
else
  echo "Enterprise Vault upgrade failed. Check the log at $LOG_FILE for details."
  exit 1
fi
=======================
#!/bin/bash

echo "Starting Post-Upgrade Checks..."

# Verify Enterprise Vault version
ev_version=$(cat /opt/enterprise_vault/version.txt)
echo "Enterprise Vault Version: $ev_version"

# Start Enterprise Vault services
echo "Starting Enterprise Vault services..."
# Replace with actual commands to start services
systemctl start enterprise_vault.service

# Verify services are running
ev_services=$(ps aux | grep '[E]nterpriseVault')
if [ -n "$ev_services" ]; then
  echo "All Enterprise Vault services are running."
  exit 0
else
  echo "Some Enterprise Vault services failed to start."
  exit 1
fi

```



``` powershell
# ev_precheck.ps1

Write-Host "Starting Pre-Upgrade Checks..."

# Check if the user has administrative privileges
If (-Not ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] "Administrator")) {
    Write-Error "You must run this script as an administrator."
    Exit 1
}

# Check available disk space (e.g., require at least 5GB free on C:)
$freeSpace = (Get-WMIObject Win32_LogicalDisk -Filter "DeviceID='C:'").FreeSpace
If ($freeSpace -lt 5GB) {
    Write-Error "Not enough free disk space on C: drive."
    Exit 1
}

# Check if Enterprise Vault services are running
$evServices = Get-Service -DisplayName "Enterprise Vault*"
$runningServices = $evServices | Where-Object {$_.Status -eq 'Running'}

If ($runningServices) {
    Write-Host "Stopping Enterprise Vault services..."
    $runningServices | Stop-Service -Force
}

Write-Host "Pre-Upgrade Checks Completed Successfully."
Exit 0

# ev_upgrade.ps1

Write-Host "Starting Enterprise Vault Upgrade..."

$installerPath = "C:\Temp\Veritas_Enterprise_Vault_Installer.exe"
$responseFile = "C:\Temp\setup.ini"
$logFile = "C:\Temp\ev_upgrade.log"

# Execute the silent upgrade
Start-Process -FilePath $installerPath -ArgumentList "/s /f1`"$responseFile`" /f2`"$logFile`"" -Wait -PassThru

# Check the installation log for success
$logContent = Get-Content $logFile
If ($logContent -match "Installation Complete") {
    Write-Host "Enterprise Vault upgrade completed successfully."
    Exit 0
} Else {
    Write-Error "Enterprise Vault upgrade failed. Check the log at $logFile for details."
    Exit 1
}

# ev_postcheck.ps1

Write-Host "Starting Post-Upgrade Checks..."

# Verify Enterprise Vault version
$evVersion = (Get-ItemProperty "HKLM:\SOFTWARE\Wow6432Node\KVS\Enterprise Vault").FileVersion
Write-Host "Enterprise Vault Version: $evVersion"

# Check if Enterprise Vault services are running
$evServices = Get-Service -DisplayName "Enterprise Vault*"
$stoppedServices = $evServices | Where-Object {$_.Status -ne 'Running'}

If ($stoppedServices) {
    Write-Host "Starting Enterprise Vault services..."
    $stoppedServices | Start-Service
}

# Verify services are running
$evServices = Get-Service -DisplayName "Enterprise Vault*"
$runningServices = $evServices | Where-Object {$_.Status -eq 'Running'}

If ($runningServices.Count -eq $evServices.Count) {
    Write-Host "All Enterprise Vault services are running."
    Exit 0
} Else {
    Write-Error "Some Enterprise Vault services failed to start."
    Exit 1
}


```

``` yaml
---
- name: Validate Semantic Versions and Upgrade Enterprise Vault
  hosts: ev_servers
  gather_facts: no
  vars:
    requested_ev_version: '14.2.2.0'  # Replace with your desired version
    semver_regex: '^(0|[1-9]\\d*)\\.(0|[1-9]\\d*)\\.(0|[1-9]\\d*)(?:-([\\da-zA-Z-]+(?:\\.[\\da-zA-Z-]+)*))?(?:\\+([\\da-zA-Z-]+(?:\\.[\\da-zA-Z-]+)*))?$'
  tasks:

    - name: Get Enterprise Vault version from registry
      win_regedit:
        path: 'HKLM:\SOFTWARE\Wow6432Node\KVS\Enterprise Vault'
        name: 'FileVersion'
        datatype: string
        state: read
      register: ev_version_info

    - name: Set current EV version fact
      set_fact:
        current_ev_version: "{{ ev_version_info.value }}"
      when: ev_version_info.value is defined

    - name: Fail if current EV version could not be retrieved
      fail:
        msg: "Could not retrieve the current Enterprise Vault version."
      when: current_ev_version is not defined

    - name: Validate that current EV version is a valid semantic version
      set_fact:
        is_current_version_valid: "{{ current_ev_version is match(semver_regex) }}"

    - name: Validate that requested EV version is a valid semantic version
      set_fact:
        is_requested_version_valid: "{{ requested_ev_version is match(semver_regex) }}"

    - name: Fail if current EV version is not a valid semantic version
      fail:
        msg: "Current EV version '{{ current_ev_version }}' is not a valid semantic version."
      when: not is_current_version_valid

    - name: Fail if requested EV version is not a valid semantic version
      fail:
        msg: "Requested EV version '{{ requested_ev_version }}' is not a valid semantic version."
      when: not is_requested_version_valid

    - name: Debug current and requested EV versions
      debug:
        msg: "Current EV Version: {{ current_ev_version }}, Requested EV Version: {{ requested_ev_version }}"

    - name: Check if requested version is greater than current version
      set_fact:
        upgrade_required: "{{ current_ev_version is version(requested_ev_version, '<') }}"

    - name: Debug version comparison result
      debug:
        msg: "Upgrade Required: {{ upgrade_required }}"

    - name: Perform EV upgrade if required
      block:

        - name: Pre-upgrade tasks
          debug:
            msg: "Performing pre-upgrade tasks..."

        - name: Run EV upgrade script
          debug:
            msg: "Running EV upgrade script..."

        - name: Post-upgrade tasks
          debug:
            msg: "Performing post-upgrade tasks..."

      when: upgrade_required

    - name: Skip upgrade as current version is up-to-date
      debug:
        msg: "Enterprise Vault is up-to-date. No upgrade required."
      when: not upgrade_required

```
