
rg="pe-proxy"
loc="eastus"
vnet="pe-proxy-vnet"
vnet_prefix="10.2.0.0/16"
subnet1="pe-subnet"
subnet2="vm-subnet"
subnet3="proxy-subnet"
subnet1_prefix="10.2.1.0/24"
subnet2_prefix="10.2.2.0/24"
subnet3_prefix="10.2.3.0/24"
nginx_private_ip="10.2.3.5"
storage="peproxy"$RANDOM
blob_container="images"
blob_name="cat.jpg"
blob_url="https://$storage.blob.core.windows.net/$blob_container/$blob_name?"

BLACK="\033[30m"
RED="\033[31m"
GREEN="\033[32m"
YELLOW="\033[33m"
BLUE="\033[34m"
PINK="\033[35m"
CYAN="\033[36m"
WHITE="\033[37m"
NORMAL="\033[0;39m"

# Allow RG to be set via shell var
if [[ $1 ]]; then
    rg=$1
fi

echo -e "$WHITE$(date +"%T")$GREEN Creating Resource Group$CYAN" $rg "$GREEN in $CYAN" $loc
az group create --name $rg --location $loc --output none

az network nsg create -g $rg -n $vnet-nsg --output none

echo -e "$WHITE$(date +"%T")$GREEN Creating$CYAN Vnet" $vnet $WHITE
echo -e "$WHITE$(date +"%T")$GREEN Creating$CYAN subnet1" $subnet1 $WHITE
az network vnet create --resource-group $rg --name $vnet --address-prefixes $vnet_prefix --subnet-name $subnet1 --subnet-prefix $subnet1_prefix --output none
echo -e "$WHITE$(date +"%T")$GREEN Creating$CYAN subnet2" $subnet2 $WHITE
az network vnet subnet create --resource-group $rg --vnet-name $vnet --name $subnet2 --address-prefixes $subnet2_prefix --output none
echo -e "$WHITE$(date +"%T")$GREEN Creating$CYAN subnet3" $subnet3 $WHITE
az network vnet subnet create --resource-group $rg --vnet-name $vnet --name $subnet3 --address-prefixes $subnet3_prefix --output none

az network vnet subnet update \
    --resource-group $rg \
    --name $subnet3 \
    --vnet-name $vnet \
    --delegations NGINX.NGINXPLUS/nginxDeployments \
    --output none

echo -e "$WHITE$(date +"%T")$GREEN Creating$CYAN storage" $storage $WHITE
az storage account create --name $storage --resource-group $rg --location $loc --sku Standard_LRS --output none
echo -e "$WHITE$(date +"%T")$GREEN Creating$CYAN container images" $WHITE
acctkey=$(az storage account keys list --account-name $storage --resource-group $rg --query '[0].value' --output tsv)
az storage container create --name $blob_container --account-name $storage --account-key $acctkey --output none

az storage blob upload \
    --account-name $storage \
    --container-name $blob_container \
    --name $blob_name \
    --file assets/$blob_name \
    --account-key $acctkey \
    --output none

if (uname -a | grep -q 'Darwin'); then
    # MacOS
    end=`date -v +30d -u '+%Y-%m-%dT%H:%MZ'`
else
    # Linux
    end=`date -u -d "30 days" '+%Y-%m-%dT%H:%MZ'`
fi

sas=$(az storage blob generate-sas \
    --account-name $storage \
    --container-name $blob_container \
    --name $blob_name \
    --permissions r \
    --expiry $end \
    --https-only \
    --account-key $acctkey \
    --output tsv)

echo -e "$WHITE$(date +"%T")$GREEN SAS Token for$CYAN storage" $storage $WHITE
echo $blob_url$sas

storage_id=$(az storage account list \
    --resource-group $rg \
    --query '[].[id]' \
    --output tsv)

az network private-endpoint create \
    --connection-name connection-1 \
    --name $rg_pe \
    --private-connection-resource-id $storage_id \
    --resource-group $rg \
    --subnet $subnet1 \
    --group-id blob \
    --vnet-name $vnet

az network private-dns zone create \
    --resource-group $rg \
    --name "privatelink.blob.core.windows.net"

az network private-dns link vnet create \
    --resource-group $rg \
    --zone-name "privatelink.blob.core.windows.net" \
    --name dns-link \
    --virtual-network $vnet \
    --registration-enabled false

az network private-endpoint dns-zone-group create \
    --resource-group $rg \
    --endpoint-name $rg_pe \
    --name default \
    --private-dns-zone "privatelink.blob.core.windows.net" \
    --zone-name privatelink-blob-core-windows-net
