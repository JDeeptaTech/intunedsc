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
