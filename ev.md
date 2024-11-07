``` sh
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
