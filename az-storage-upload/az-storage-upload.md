# How to upload to Azure Storage from Linux machine just with curl

## Steps

1. Create Azure Storage account and container to receive the uploaded files
2. Issue SAS token for the container allowing restricted upload access


You may do all preparation in [Azure Shell](https://shell.azure.com) 
or in your local machine with Azure CLI (`az``).

## Prepare the storage account

```bash
#!/bin/bash

# Generate a unique storage account name
generate_storage_account_name() {
    local prefix="up"  # Change this to your desired prefix
    local timestamp=$(date +%Y%m%d%H%M%S)
    local random_part=$((RANDOM % 10000))  # Change range as needed
    echo "${prefix}${timestamp}${random_part}"
}

storage_account_name=$(generate_storage_account_name)

RESOURCE_GROUP_NAME="$storage_account_name"
STORAGE_ACCOUNT_NAME="$storage_account_name"
CONTAINER_NAME=upload

# note for later reference
echo
echo 'Storage Account details:'
echo RESOURCE_GROUP_NAME="$storage_account_name"
echo STORAGE_ACCOUNT_NAME="$storage_account_name"
echo CONTAINER_NAME=upload
echo 

# Create resource group
az group create --name $RESOURCE_GROUP_NAME --location westeurope

# Create storage account
az storage account create --resource-group $RESOURCE_GROUP_NAME --name $STORAGE_ACCOUNT_NAME --sku Standard_LRS --encryption-services blob

# Create blob container
az storage container create --name $CONTAINER_NAME --account-name $STORAGE_ACCOUNT_NAME --public-access off

# note for later reference
echo
echo 'Storage Account details:'
echo RESOURCE_GROUP_NAME="$storage_account_name"
echo STORAGE_ACCOUNT_NAME="$storage_account_name"
echo CONTAINER_NAME=upload
echo 

```

## Create SAS token for upload operation

Notice that SAS token has specific time validity, permissions and other restrictions like source ip.

```bash
#!/bin/bash

# upload access valid only for next 1 hour
expiry=$(date -u -d "+1 hour" "+%Y-%m-%dT%H:%M:%SZ")

# generate the SAS token for upload
sas_token=$(az storage container generate-sas --account-name "$STORAGE_ACCOUNT_NAME" --name "$CONTAINER_NAME" --permissions cw --expiry "$expiry" --output tsv)

# Print the SAS token
echo
echo "Generated SAS Token: $sas_token"
```

## Share upload instruction (command)

```bash

file_name="yourfile.zip"

echo
echo "Upload URL: 'https://$STORAGE_ACCOUNT_NAME.blob.core.windows.net/$CONTAINER_NAME/$file_name?$sas_token'"

echo
echo "Upload command:"
echo "curl -v -T $file_name" '-H "x-ms-date: $(date -u)"' '-H "x-ms-blob-type: BlockBlob"' "'""https://$STORAGE_ACCOUNT_NAME.blob.core.windows.net/$CONTAINER_NAME/$file_name?$sas_token""'"
```


### Whole upload command generation script

Update file_name and Storage Account details below before running the script.
Script is designed to run in Azure Shell.

It will generate file upload command for you to be execurted on Linux server uploading the file.
It depends on `curl` command only.

```bash
#!/bin/bash

# filename
file_name="yourfile.zip"

# update with real data
RESOURCE_GROUP_NAME=up202308161148326358
STORAGE_ACCOUNT_NAME=up202308161148326358
CONTAINER_NAME=upload
 
# upload access valid only for next 1 hour
expiry=$(date -u -d "+1 hour" "+%Y-%m-%dT%H:%M:%SZ")

# generate the SAS token for upload
sas_token=$(az storage container generate-sas --account-name "$STORAGE_ACCOUNT_NAME" --name "$CONTAINER_NAME" --permissions cw --expiry "$expiry" --output tsv)

echo "Upload URL: 'https://$STORAGE_ACCOUNT_NAME.blob.core.windows.net/$CONTAINER_NAME/$file_name?$sas_token'"

echo
echo "Upload command:"
echo "curl -v -T $file_name" '-H "x-ms-date: $(date -u)"' '-H "x-ms-blob-type: BlockBlob"' "'""https://$STORAGE_ACCOUNT_NAME.blob.core.windows.net/$CONTAINER_NAME/$file_name?$sas_token""'"
echo

```

### Similar from Powershell

Once we know how to get the SAS token, we can use it also to upload from Powershell:

```powershell
$filename = "a.md";
$sas_token = 'se=2023-08-16T13%3A34%3A52Z&sp=cw&sv=2022-11-02&sr=c&sig=lbxofmYr5XALHXHi0nCdBpCXa0HDdGC/1bSt1QKEvmw%3D'
Invoke-RestMethod -Uri ('https://up202308161148326358.blob.core.windows.net/upload/' + $filename + '?' + $sas_token) -InFile $filename -Method PUT -Headers @{"x-ms-blob-type" = "BlockBlob"}
```

