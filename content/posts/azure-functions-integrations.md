az group create \
    --name balder-functions \
    --location westeurope

az storage account create \
    --resource-group balder-functions \
    --name balderxenitfunctest \
    --sku Standard_LRS

az functionapp plan create \
    --resource-group balder-functions \
    --location westeurope \
    --name balder-functions \
    --number-of-workers 1 \
    --sku EP1

az functionapp create \
    --resource-group balder-functions \
    --name balder-function-1 \
    --plan balder-functions \
    --storage-account balderxenitfunctest \
    --functions-version 3 \
    --runtime node \
    --runtime-version 14

az storage account show-connection-string \
    --resource-group balder-functions \
    --name balderxenitfunctest \
    --query connectionString \
    --output tsv

az functionapp config appsettings set \
    --resource-group balder-functions \
    --name balder-function-1 \
    --settings 'AzureWebJobsStorage=DefaultEndpointsProtocol=https;EndpointSuffix=core.windows.net;AccountName=balderxenitfunctest;AccountKey=LZ5RkVn0RvTtUTqiCwYZICQFB/Ro/bLh5rt8PJ5GL0b26nN0ZeCjWValJYaCaZM9VcjfnPey1kPvO6Bb7vEcIw=='

az functionapp config appsettings set \
    --resource-group balder-functions \
    --name balder-function-1 \
    --settings UPM_URL=https://ifconfig.co

az functionapp config appsettings set \
    --resource-group balder-functions \
    --name balder-function-1 \
    --settings DYNAMICS_URL=https://ingress-healthz.prod.unbox.xenit.io/
