# intunedsc

trigger:
  branches:
    include:
      - main # Adjust as per your branch strategy

pool:
  vmImage: 'windows-latest'

variables:
  ConfigurationFolder: 'Configurations' # Folder where the configuration scripts are stored

steps:
# Step 1: Install PowerShell modules required (Microsoft365DSC, etc.)
- task: PowerShell@2
  displayName: 'Install Microsoft365DSC Module'
  inputs:
    targetType: 'inline'
    script: |
      Install-Module -Name Microsoft365DSC -Force -AllowClobber

# Step 2: Compile each configuration script in the Configurations folder into .mof files
- task: PowerShell@2
  displayName: 'Compile DSC Configurations to MOF'
  inputs:
    targetType: 'inline'
    script: |
      $configFolder = "$(ConfigurationFolder)"
      $outputFolder = "$(Build.ArtifactStagingDirectory)\MOF"

      # Create output folder if it doesn't exist
      if (-not (Test-Path $outputFolder)) {
        New-Item -Path $outputFolder -ItemType Directory
      }

      # Get all .ps1 configuration files in the configurations folder
      $configFiles = Get-ChildItem -Path $configFolder -Filter *.ps1

      foreach ($configFile in $configFiles) {
        Write-Host "Processing configuration file: $($configFile.FullName)"
        
        # Execute each configuration file to create the DSC configurations
        . $configFile.FullName

        # Get the configuration function name from the filename (assumed convention)
        $configFunctionName = $configFile.BaseName

        # Call the configuration function to generate MOF files
        # MOF files will be created in a folder matching the function name
        Start-DscConfiguration -Path .\$configFunctionName -OutputPath $outputFolder -Verbose -Wait -Force

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
