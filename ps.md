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

```ps
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
