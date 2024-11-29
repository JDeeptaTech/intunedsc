```yaml
- task: AzurePowerShell@5
  inputs:
    azureSubscription: 'YourServiceConnectionName'
    ScriptType: 'InlineScript'
    Inline: |
      # Variables
      $resourceGroupName = "YourResourceGroupName"
      $virtualNetworkName = "YourVNetName"
      $subnetName = "YourSubnetName"
      $subnetPrefix = "10.0.1.0/24"

      # Get the virtual network
      $virtualNetwork = Get-AzVirtualNetwork -Name $virtualNetworkName -ResourceGroupName $resourceGroupName

      if (-not $virtualNetwork) {
          Write-Error "Virtual network '$virtualNetworkName' not found in resource group '$resourceGroupName'."
          exit 1
      }

      # Check if the subnet exists
      $subnet = $virtualNetwork.Subnets | Where-Object { $_.Name -eq $subnetName }

      if (-not $subnet) {
          # Subnet does not exist, create it
          $virtualNetwork | Add-AzVirtualNetworkSubnetConfig -Name $subnetName -AddressPrefix $subnetPrefix | Set-AzVirtualNetwork
          Write-Host "Subnet '$subnetName' has been created in virtual network '$virtualNetworkName'."
      } else {
          Write-Host "Subnet '$subnetName' already exists in virtual network '$virtualNetworkName'."
      }
    azurePowerShellVersion: 'LatestVersion'  # Use the latest version of Azure PowerShell


```

```yml
# Azure DevOps Pipeline YAML

trigger:
- main  # or your desired branch

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: AzureCLI@2
  displayName: 'Create Subnet if it Does Not Exist'
  inputs:
    azureSubscription: 'YourAzureServiceConnection'  # Replace with your service connection name
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      # Variables
      RESOURCE_GROUP='myResourceGroup'   # Replace with your resource group name
      VNET_NAME='myVNet'                 # Replace with your virtual network name
      SUBNET_NAME='mySubnet'             # Replace with your subnet name
      ADDRESS_PREFIX='10.0.0.0/24'       # Replace with your address prefix

      # Check if the subnet exists
      if ! az network vnet subnet show \
           --resource-group $RESOURCE_GROUP \
           --vnet-name $VNET_NAME \
           --name $SUBNET_NAME >/dev/null 2>&1; then
        echo "Subnet '$SUBNET_NAME' does not exist. Creating..."
        # Create the subnet
        az network vnet subnet create \
          --resource-group $RESOURCE_GROUP \
          --vnet-name $VNET_NAME \
          --name $SUBNET_NAME \
          --address-prefixes $ADDRESS_PREFIX
        echo "Subnet '$SUBNET_NAME' has been created."
      else
        echo "Subnet '$SUBNET_NAME' already exists. No action needed."
      fi

```
