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
