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

variables:
  ConfigurationFolder: 'Configurations'
  TenantId: $(TenantId)
  ClientId: $(ClientId)
  ClientSecret: $(ClientSecret) # Secret variable

steps:
# Step 1: Install the necessary modules
- task: PowerShell@2
  displayName: 'Install Microsoft365DSC Module'
  inputs:
    targetType: 'inline'
    script: |
      Install-Module -Name Microsoft365DSC -Force -AllowClobber

# Step 2: Create a PSCredential object and pass to DSC
- task: PowerShell@2
  displayName: 'Compile DSC Configurations to MOF with Credentials'
  inputs:
    targetType: 'inline'
    script: |
      $configFolder = "$(ConfigurationFolder)"
      $outputFolder = "$(Build.ArtifactStagingDirectory)\MOF"

      # Create output folder if it doesn't exist
      if (-not (Test-Path $outputFolder)) {
        New-Item -Path $outputFolder -ItemType Directory
      }

      # Create a PSCredential object using the pipeline variables
      $securePassword = ConvertTo-SecureString $(ClientSecret) -AsPlainText -Force
      $AdminCredential = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $(ClientId), $securePassword

      # Get all .ps1 configuration files in the configurations folder
      $configFiles = Get-ChildItem -Path $configFolder -Filter *.ps1

      foreach ($configFile in $configFiles) {
        Write-Host "Processing configuration file: $($configFile.FullName)"
        
        # Execute each configuration file and pass the credentials
        . $configFile.FullName

        # Get the configuration function name from the filename (assumed convention)
        $configFunctionName = $configFile.BaseName

        # Call the configuration function to generate MOF files
        Start-DscConfiguration -Path .\$configFunctionName -OutputPath $outputFolder -Verbose -Wait -Force -Credential $AdminCredential

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
