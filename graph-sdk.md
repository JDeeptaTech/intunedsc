``` powershell
# 1. Install the Microsoft Graph PowerShell SDK if not already installed
if (-not (Get-Module -ListAvailable -Name Microsoft.Graph)) {
    Install-Module Microsoft.Graph -Scope CurrentUser -Force
}

# Import the module
Import-Module Microsoft.Graph

# 2. Authenticate to Microsoft Graph with the necessary scopes
$Scopes = @("DeviceManagementConfiguration.ReadWrite.All")
Connect-MgGraph -Scopes $Scopes

# Check the connection status
if ((Get-MgContext).Account -eq $null) {
    Write-Host "Authentication failed. Please check your credentials." -ForegroundColor Red
    exit
}

# 3. Define the Android device configuration policy parameters
$androidPolicy = @{
    "@odata.type" = "#microsoft.graph.androidGeneralDeviceConfiguration"
    DisplayName = "Sample Android Device Configuration Policy"
    Description = "This policy enforces password requirements and security settings on Android devices."
    PasswordRequired = $true
    PasswordMinimumLength = 6
    PasswordRequiredType = "alphanumeric"  # Options: deviceDefault, alphabetic, alphanumeric, numeric, unknown
    PasswordMinutesOfInactivityBeforeScreenTimeout = 5
    CameraBlocked = $false
    FactoryResetBlocked = $false
    ScreenCaptureBlocked = $true
}

# 4. Create the policy in Intune
try {
    $newPolicy = New-MgDeviceManagementDeviceConfiguration -BodyParameter $androidPolicy
    Write-Host "Policy created successfully with ID: $($newPolicy.Id)" -ForegroundColor Green
} catch {
    Write-Host "An error occurred while creating the policy: $_" -ForegroundColor Red
}

# 5. Disconnect from Microsoft Graph
Disconnect-MgGraph

```
