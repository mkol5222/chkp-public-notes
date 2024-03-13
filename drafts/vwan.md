
```shell
# all resources - to see vwan and rg
az resource list --query "[?type=='Microsoft.Network/virtualWans']" -o table

az resource list --resource-type Microsoft.Network/virtualWans -o table

# Microsoft.Solutions/applications
az resource list --resource-type Microsoft.Solutions/applications -o table
# for
# az rest --method POST --uri "/subscriptions/<SUBSCIRPTION ID>/resourceGroups/<RG_NAME>/providers/Microsoft.Solutions/applications/<MANAGED APP>/refreshPermissions?api-version=2019-07-01&targetVersion=1.0.7"
az rest --method GET --uri "https://management.azure.com/subscriptions/f4ad5e85-ec75-4321-8854-ed7eb611f61d/resourceGroups/poc-vwan-nva/providers/Microsoft.Solutions/applications?api-version=2019-07-01"

az rest --method GET --uri "https://management.azure.com/subscriptions/f4ad5e85-ec75-4321-8854-ed7eb611f61d/resourceGroups/poc-vwan-nva/providers/Microsoft.Solutions/applications/cgns?api-version=2019-07-01"

az rest --method POST --uri 'https://management.azure.com/subscriptions/f4ad5e85-ec75-4321-8854-ed7eb611f61d/resourceGroups/poc-vwan-nva/providers/Microsoft.Solutions/applications/cgns/refreshPermissions?api-version=2019-07-01&targetVersion=1.0.7'


az resource list   -o json | ConvertFrom-Json | group-object -Property type | select name,count

# list vwan
az network vwan list -o table

# show my hub
az network vhub list -o table
```

REST API with `az rest`:

```shell
# get default subscription 
az account show --query id -o tsv
az account show -o table
# put it in URL below to list RGs
az rest --uri https://management.azure.com/subscriptions/f4ad5e85-ec75-4321-8854-ed7eb611f61d/resourcegroups?api-version=2020-06-01

az rest --uri https://management.azure.com/subscriptions/f4ad5e85-ec75-4321-8854-ed7eb611f61d/resourceGroups/poc-vwan-nva/providers/Microsoft.Resources/deployments/checkpoint.azure-vwan-20240104085745?api-version=2020-06-01

az rest --uri https://management.azure.com/subscriptions/f4ad5e85-ec75-4321-8854-ed7eb611f61d/resourceGroups/poc-vwan-nva/providers/Microsoft.Resources/deployments/checkpoint.azure-vwan-20240104085745?api-version=2020-06-01 | ConvertFrom-Json | select-object -ExpandProperty properties | select-object -ExpandProperty parameters | ft imageVersion, location

```

```shell
```

```shell
```