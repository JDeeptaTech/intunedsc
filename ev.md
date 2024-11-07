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
- name: Upgrade Veritas Enterprise Vault
  hosts: ev_servers
  become: yes
  vars:
    installer_source: "/network/share/Veritas_Enterprise_Vault_Installer.exe"
    response_file_source: "/network/share/setup.ini"
    installer_dest: "/tmp/Veritas_Enterprise_Vault_Installer.exe"
    response_file_dest: "/tmp/setup.ini"

  tasks:
    - name: Copy the installer to the target server
      copy:
        src: "{{ installer_source }}"
        dest: "{{ installer_dest }}"
        mode: '0755'

    - name: Copy the response file to the target server
      copy:
        src: "{{ response_file_source }}"
        dest: "{{ response_file_dest }}"
        mode: '0644'

    - name: Copy the upgrade script to the target server
      copy:
        src: "ev_upgrade.sh"
        dest: "/tmp/ev_upgrade.sh"
        mode: '0755'

    - name: Execute the upgrade script
      shell: "/tmp/ev_upgrade.sh"
      args:
        executable: /bin/bash

    - name: Cleanup temporary files
      file:
        path: "{{ item }}"
        state: absent
      loop:
        - "{{ installer_dest }}"
        - "{{ response_file_dest }}"
        - "/tmp/ev_upgrade.sh"

```
