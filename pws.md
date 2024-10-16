``` powershell
# Define the root folder containing subfolders with partial configurations
$RootFolder = "C:\Path\To\Your\Configs"

# Output folder for the generated MOF files
$OutputFolder = "C:\Path\To\MOFOutput"

# Import the Microsoft365DSC module
Import-Module Microsoft365DSC

# Get all partial configuration scripts from the subfolders
$PartialConfigs = Get-ChildItem -Path $RootFolder -Recurse -Filter *.ps1

# Iterate through each partial configuration file and compile them
foreach ($Config in $PartialConfigs) {
    Write-Host "Processing partial configuration: $($Config.FullName)"

    # Source the partial configuration script
    . $Config.FullName

    # Extract the configuration name (assuming it's in the form 'ConfigurationName { ... }' within the script)
    $ConfigContent = Get-Content $Config.FullName -Raw
    if ($ConfigContent -match 'Configuration\s+(\w+)\s*\{') {
        $ConfigName = $matches[1]
        
        # Dynamically compile the configuration and generate the MOF file
        Write-Host "Compiling configuration: $ConfigName"

        # Ensure output folder exists
        if (!(Test-Path $OutputFolder)) {
            New-Item -Path $OutputFolder -ItemType Directory
        }

        # Compile and generate MOF
        $OutputPath = Join-Path -Path $OutputFolder -ChildPath $ConfigName
        Start-DscConfiguration -Path $OutputPath -Verbose -Wait -Force

        Write-Host "MOF file generated at: $OutputPath"
    } else {
        Write-Host "Could not find configuration block in: $($Config.FullName)"
    }
}

Write-Host "MOF generation complete."

```

``` powershell
# Use a base Windows Server Core image for Windows containers (or Ubuntu for Linux containers)
FROM mcr.microsoft.com/windows/servercore:ltsc2022

# Set environment variables for the agent
ENV AGENT_VERSION=3.220.2

# Install PowerShell and other dependencies
RUN powershell -Command \
    Set-ExecutionPolicy Bypass -Scope Process -Force; \
    Install-Module -Name PowerShellGet -Force; \
    Install-Module -Name Microsoft365DSC -Force -AllowClobber

# Download and extract the Azure Pipelines agent
RUN powershell -Command \
    Invoke-WebRequest -Uri https://vstsagentpackage.azureedge.net/agent/$env:AGENT_VERSION/vsts-agent-win-x64-$env:AGENT_VERSION.zip -OutFile agent.zip; \
    Expand-Archive -Path agent.zip -DestinationPath C:\agent; \
    Remove-Item -Force agent.zip

# Set up agent directory
WORKDIR C:/agent

# Install Azure DevOps agent
RUN powershell -Command \
    ./config.cmd --unattended --url https://dev.azure.com/YOUR_ORG --auth PAT --token YOUR_PAT --pool default --agent AGENT_NAME --work _work --replace

# Start the agent in interactive mode
CMD ./run.cmd

```

``` powershell
# Import Required Modules
Import-Module Microsoft365DSC
Import-Module powershell-yaml

# Load YAML Configuration
$configFilePath = "./config.yml"
$configData = ConvertFrom-Yaml (Get-Content $configFilePath -Raw)

# Define Output Path for the Generated Configuration
$outputPath = "./GeneratedConfig.ps1"

# Start building the configuration file
$generatedConfig = @"
# Auto-generated Microsoft365DSC Configuration
Configuration MyGeneratedConfig {
    param (
        [PSCredential]`$Creds
    )

    Import-DscResource -ModuleName Microsoft365DSC

    Node "localhost" {
"@

# Add SharePoint Site Collection Configurations
foreach ($siteCollection in $configData.sharepoint.siteCollections) {
    $generatedConfig += @"
        SPO_SiteCollection "SiteCollection_$($siteCollection.title)" {
            Url              = "$($siteCollection.url)"
            Owner            = "$($siteCollection.owner)"
            StorageQuota     = $($siteCollection.storageQuota)
            Template         = "$($siteCollection.template)"
            Title            = "$($siteCollection.title)"
            Ensure           = "Present"
            Credential       = `$Creds
        }
"@
}

# Add SharePoint Tenant Settings
$generatedConfig += @"
        SPO_TenantSettings "SharePointTenantSettings" {
            CommentsOnSitePagesDisabled = $($configData.sharepoint.tenantSettings.commentsOnSitePagesDisabled)
            ExternalSharingAllowed      = $($configData.sharepoint.tenantSettings.externalSharingAllowed)
            Credential                  = `$Creds
        }
"@

# Add Teams Channel Configurations
foreach ($teamChannel in $configData.teams.teamChannels) {
    $generatedConfig += @"
        Teams_TeamChannel "Channel_$($teamChannel.channelName)" {
            TeamName    = "$($teamChannel.teamName)"
            DisplayName = "$($teamChannel.channelName)"
            Credential  = `$Creds
            Ensure      = "Present"
        }
"@
}

# Add Teams Tenant Settings
$generatedConfig += @"
        Teams_TenantSettings "TeamsTenantSettings" {
            AllowGuestUser = $($configData.teams.tenantSettings.allowGuestUser)
            Credential     = `$Creds
        }
"@

# Close Configuration block
$generatedConfig += @"
    }
}
"@

# Write the generated configuration to a file
$generatedConfig | Out-File -FilePath $outputPath -Force

# Output to console for visibility
Write-Host "Configuration generated at $outputPath"
Write-Host "`nGenerated Configuration:"
Write-Host $generatedConfig

# Optionally, apply the generated configuration immediately
$applyConfig = Read-Host -Prompt "Would you like to apply the generated configuration now? (Y/N)"
if ($applyConfig -eq 'Y') {
    $creds = Get-Credential -Message "Please provide Microsoft 365 admin credentials"
    . $outputPath
    Start-DscConfiguration -Path ./MyGeneratedConfig -Wait -Force -Verbose
}

```

``` powershell
# Helper function to merge two objects, overriding common properties
function Merge-Objects {
    param (
        [PSCustomObject]$PrimaryObject,
        [PSCustomObject]$OverrideObject
    )
    
    # Clone the primary object to avoid modifying it directly
    $MergedObject = $PrimaryObject.PSObject.Copy()

    # Loop through each property in the override object
    $OverrideObject | Get-Member -MemberType Properties | ForEach-Object {
        $propertyName = $_.Name
        
        # If the property exists in both objects, override it
        if ($MergedObject.PSObject.Properties.Match($propertyName)) {
            $MergedObject | Add-Member -MemberType NoteProperty -Name $propertyName -Value $OverrideObject.$propertyName -Force
        }
    }
    
    return $MergedObject
}

# Merge obj1 and obj2, properties in obj2 override obj1
$mergedObject = Merge-Objects -PrimaryObject $obj1 -OverrideObject $obj2

# Output the merged object
$mergedObject

```
