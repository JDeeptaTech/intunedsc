# intunedsc

```powershell
Configuration IntuneCompliancePolicy
{
    param (
        [PSCredential]$AdminCredential
    )

    Import-DscResource -ModuleName Microsoft365DSC

    Node "localhost"
    {
        # Example: Configuring Intune device compliance policy for Windows 10
        IntuneDeviceCompliancePolicyWindows10_DeviceCompliancePolicy {
            DisplayName         = "Simple Windows 10 Compliance Policy"
            Ensure              = "Present"
            AdminCredential     = $AdminCredential
            RequireSecureBoot   = $true
            OsMinimumVersion    = "10.0.19041"
            OsMaximumVersion    = "10.0.99999"
            PasswordRequired    = $true
        }
    }
}

# Run the configuration
$AdminCredential = Get-Credential
IntuneCompliancePolicy -AdminCredential $AdminCredential

```

``` yaml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'windows-latest'

# Reference the variable group (e.g., M365Credentials)
variables:
  - group: M365Credentials

steps:
# Step 1: Install the necessary PowerShell modules
- task: PowerShell@2
  displayName: 'Install Microsoft365DSC Module'
  inputs:
    targetType: 'inline'
    script: |
      Install-Module -Name Microsoft365DSC -Force -AllowClobber

# Step 2: Create a PSCredential object and compile the DSC configuration into MOF files
- task: PowerShell@2
  displayName: 'Compile DSC Configurations to MOF with Credentials'
  inputs:
    targetType: 'inline'
    script: |
      $configFolder = "$(Build.SourcesDirectory)\Configurations"
      $outputFolder = "$(Build.ArtifactStagingDirectory)\MOF"

      # Create output folder if it doesn't exist
      if (-not (Test-Path $outputFolder)) {
        New-Item -Path $outputFolder -ItemType Directory
      }

      # Create a PSCredential object using the variables from the variable group
      $securePassword = ConvertTo-SecureString "$(ClientSecret)" -AsPlainText -Force
      $AdminCredential = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList "$(ClientId)", $securePassword

      # Get all .ps1 configuration files in the configurations folder
      $configFiles = Get-ChildItem -Path $configFolder -Filter *.ps1

      foreach ($configFile in $configFiles) {
        Write-Host "Processing configuration file: $($configFile.FullName)"
        
        # Source the configuration file to define the configuration function
        . $configFile.FullName

        # Get the configuration function name from the filename (assuming it matches the file name)
        $configFunctionName = $configFile.BaseName

        # Call the configuration function to generate MOF files
        # Use the output folder for MOF file storage
        $configFunctionName -AdminCredential $AdminCredential -OutputPath $outputFolder

        Write-Host "MOF files generated for $configFunctionName."
      }

      # List the MOF files generated
      Get-ChildItem -Path $outputFolder -Filter *.mof | ForEach-Object { Write-Host "Generated MOF file: $_" }

- task: PowerShell@2
  displayName: 'Compile DSC Configurations to MOF with Credentials'
  inputs:
    targetType: 'inline'
    script: |
      $configFolder = "$(Build.SourcesDirectory)\Configurations"
      $outputFolder = "$(Build.ArtifactStagingDirectory)\MOF"

      # Create output folder if it doesn't exist
      if (-not (Test-Path $outputFolder)) {
        New-Item -Path $outputFolder -ItemType Directory
      }

      # Create a PSCredential object using the variables from the variable group
      $securePassword = ConvertTo-SecureString "$(ClientSecret)" -AsPlainText -Force
      $AdminCredential = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList "$(ClientId)", $securePassword

      # Get all .ps1 configuration files in the configurations folder
      $configFiles = Get-ChildItem -Path $configFolder -Filter *.ps1

      foreach ($configFile in $configFiles) {
        Write-Host "Processing configuration file: $($configFile.FullName)"
        
        # Source the configuration script (this defines the configuration)
        . $configFile.FullName

        # Get the configuration function name from the filename (assuming it matches the file name)
        $configFunctionName = $configFile.BaseName

        # Call the configuration function directly
        Write-Host "Invoking configuration: $configFunctionName"
        Invoke-Expression "$configFunctionName -AdminCredential `$AdminCredential -OutputPath `$outputFolder"

        Write-Host "MOF files generated for $configFunctionName."
      }

      # List the MOF files generated
      Get-ChildItem -Path $outputFolder -Filter *.mof | ForEach-Object { Write-Host "Generated MOF file: $_" }


# Step 3: Publish the MOF files as pipeline artifacts
- task: PublishBuildArtifacts@1
  displayName: 'Publish MOF Files'
  inputs:
    PathtoPublish: '$(Build.ArtifactStagingDirectory)\MOF'
    ArtifactName: 'MOFFiles'
    publishLocation: 'Container'


```
