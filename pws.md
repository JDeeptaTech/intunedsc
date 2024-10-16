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
