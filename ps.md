```powershell
# Specify the directory to search for .zip files
$SourceDirectory = "C:\Path\To\Search"

# Specify the destination folder where the zip files will be extracted
$DestinationDirectory = "C:\Path\To\Extract"

# Ensure the destination directory exists
if (-not (Test-Path -Path $DestinationDirectory)) {
    New-Item -ItemType Directory -Path $DestinationDirectory -Force
}

# Get all .zip files in the source directory and its subdirectories
$ZipFiles = Get-ChildItem -Path $SourceDirectory -Filter "*.zip" -File -Recurse

if ($ZipFiles.Count -eq 0) {
    Write-Host "No .zip files found in the directory: $SourceDirectory"
    return
}

# Process each .zip file
foreach ($ZipFile in $ZipFiles) {
    Write-Host "Extracting $($ZipFile.FullName)..."

    # Define the extraction path (subfolder for each .zip file)
    $ExtractPath = Join-Path -Path $DestinationDirectory -ChildPath $ZipFile.BaseName

    # Ensure the extraction path exists
    if (-not (Test-Path -Path $ExtractPath)) {
        New-Item -ItemType Directory -Path $ExtractPath -Force
    }

    # Extract the .zip file using Expand-Archive
    try {
        Expand-Archive -Path $ZipFile.FullName -DestinationPath $ExtractPath -Force
        Write-Host "Successfully extracted: $($ZipFile.Name) to $ExtractPath"
    } catch {
        Write-Host "Failed to extract $($ZipFile.FullName): $_" -ForegroundColor Red
    }
}

```

```powershell
# Import both .psd1 files
$file1 = Import-PowerShellDataFile -Path "File1.psd1"
$file2 = Import-PowerShellDataFile -Path "File2.psd1"

# Initialize a new hashtable to store the merged data
$mergedData = @{}

# Copy all unique keys from both files
foreach ($key in $file1.Keys + $file2.Keys | Sort-Object -Unique) {
    if ($file1.ContainsKey($key) -and $file2.ContainsKey($key)) {
        # If both files contain the same key, merge their values
        $mergedData[$key] = $file1[$key] + $file2[$key]
    }
    elseif ($file1.ContainsKey($key)) {
        # If only File1 contains the key, add it
        $mergedData[$key] = $file1[$key]
    }
    elseif ($file2.ContainsKey($key)) {
        # If only File2 contains the key, add it
        $mergedData[$key] = $file2[$key]
    }
}

# Output the merged data (for verification)
$mergedData

# (Optional) Save the merged data back to a new .psd1 file
@"
`@{
$($mergedData | Out-String)
}
"@ | Set-Content -Path "MergedFile.psd1"

```

```powershell
# Script: Setup-Module.ps1
# Purpose: This script sets up and updates the Microsoft365DSC module along with its dependencies.
# Author: [Your Name]
# Date: [Insert Date]

# Get user credentials for repository access
$cred = Get-Credential

# Define variables
$Version = "1.24.1120.1"  # Target version of the Microsoft365DSC module
$RepoName = "nexus"       # Repository name
$RepoURL = "https://nexus302.systems.uk.hsbc:8081/nexus/repository/nuget_proxy_powershellgallery_iq/"
$M365DscPaths = @("C:\Program Files\WindowsPowerShell\Modules\Microsoft365DSC")

# Set TLS 1.2 as the security protocol
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12

# Define credentials for authentication
$user = $ENV:USER_NAME    # Replace with actual username if needed
$password = ConvertTo-SecureString $ENV:USER_PASSWORD -AsPlainText -Force
$cred = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $user, $password

# Check if the PSRepository already exists
$repo = Get-PSRepository | Where-Object { $_.Name -eq $RepoName }
if ($repo) {
    Write-Host "Repository '$RepoName' already exists."
} else {
    Write-Host "Adding repository '$RepoName'..."
    Register-PSRepository -Name $RepoName `
        -SourceLocation $RepoURL `
        -PublishLocation $RepoURL `
        -InstallationPolicy Trusted `
        -Credential $cred `
        -PackageManagementProvider NuGet
}

# Check for installed version of the module
$localModule = Get-Module Microsoft365DSC -ListAvailable | Sort-Object Version -Descending | Select-Object -First 1

# Check if the installed version matches the required version
if ($null -eq $localModule -or $localModule.Version -ne $Version) {
    Write-Host "Updating Microsoft365DSC module to version $Version..."

    # Remove old versions of the module and related dependencies
    foreach ($M365DscPath in $M365DscPaths) {
        if (Test-Path -Path $M365DscPath) {
            Write-Host "Removing old versions of Microsoft365DSC from $M365DscPath..."
            Remove-Item -Path $M365DscPath -Force -Recurse -Verbose
        }
    }

    # Install the required version of the Microsoft365DSC module
    Install-Module -Name Microsoft365DSC -RequiredVersion $Version `
        -AllowClobber -Confirm:$false -Force `
        -Repository $RepoName -Credential $cred -ErrorAction SilentlyContinue

    # Import the manifest to handle dependencies
    $manifest = Import-PowerShellDataFile -Path "C:\Program Files\WindowsPowerShell\Modules\Microsoft365DSC\$($Version)\Dependencies\Manifest.psd1"

    # Install dependencies
    foreach ($dependency in $manifest.Dependencies) {
        Write-Host "Installing dependency '$($dependency.ModuleName)' version $($dependency.RequiredVersion)..."
        Install-Module -Name $dependency.ModuleName -RequiredVersion $dependency.RequiredVersion `
            -AllowClobber -Confirm:$false -Force `
            -Repository $RepoName -Credential $cred
    }

} else {
    Write-Host "Microsoft365DSC version $Version is already installed."
}

# Check for related modules' versions
foreach ($M365DscPath in $M365DscPaths) {
    if (Test-Path -Path $M365DscPath) {
        $relatedModules = Import-PowerShellDataFile -Path "$($M365DscPath)\$($localModule.Version)\Dependencies\Manifest.psd1"
        foreach ($relatedModule in $relatedModules.Dependencies) {
            $module = Get-Module -Name $relatedModule.ModuleName -ListAvailable
            if ($module.Version -ne $relatedModule.RequiredVersion) {
                Write-Host "Updating related module '$($relatedModule.ModuleName)'..."
                Install-Module -Name $relatedModule.ModuleName -RequiredVersion $relatedModule.RequiredVersion `
                    -AllowClobber -Confirm:$false -Force `
                    -Repository $RepoName -Credential $cred
            } else {
                Write-Host "Related module '$($relatedModule.ModuleName)' version $($relatedModule.RequiredVersion) is already installed."
            }
        }
    }
}

# Optional: Set MaxEnvelopeSizeKB and proxy settings
Write-Host "Setting MaxEnvelopeSizeKB to 8192..."
Invoke-Command -ScriptBlock { cd C:\; winrm set winrm/config '@{MaxEnvelopeSizeKB="8192"}' }

Write-Host "Setting Internet proxy settings for account LocalSystem..."
& 'C:\Windows\System32\bitsadmin.exe' /util /SetIEProxy LocalSystem Manual_proxy "proxy.a2:3128" "<local>"
```