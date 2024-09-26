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
