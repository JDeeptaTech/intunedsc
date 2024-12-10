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
``` powershell
function Example-Script {
    [CmdletBinding()]
    param (
        [Parameter(ValueFromPipeline)]
        [string[]]$InputData
    )

    # Global variable to store exceptions
    $global:ExceptionLog = @()

    begin {
        Write-Host "Begin block: Initialization" -ForegroundColor Green
        # Initialize any resources or variables
        $ProcessedItems = @()
    }

    process {
        Write-Host "Process block: Processing each item" -ForegroundColor Cyan
        foreach ($item in $InputData) {
            try {
                if ($item -eq "error") {
                    throw "Simulated error for input: $item"
                }
                # Simulate processing
                Write-Host "Processing item: $item"
                $ProcessedItems += $item
            }
            catch {
                # Log exceptions
                $global:ExceptionLog += $_
                Write-Host "Caught exception for item: $item. Error: $_" -ForegroundColor Red
            }
        }
    }

    end {
        Write-Host "End block: Finalizing script execution" -ForegroundColor Yellow

        # Summarize processed items
        Write-Host "Processed Items: $($ProcessedItems -join ', ')"

        # Log any exceptions captured
        if ($global:ExceptionLog.Count -gt 0) {
            Write-Host "Exceptions captured during execution:" -ForegroundColor Red
            foreach ($exception in $global:ExceptionLog) {
                Write-Host $exception
            }
        } else {
            Write-Host "No exceptions encountered." -ForegroundColor Green
        }
    }
}

# Example usage
"item1", "item2", "error", "item3" | Example-Script

```
