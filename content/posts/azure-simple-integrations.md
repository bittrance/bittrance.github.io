```bash
az group create \
    --name logic-test \
    --location westeurope

az group show \
    --name logic-test \
    --query id \
    --output tsv
```
```bash
az monitor log-analytics workspace create \
    --resource-group logic-test \
    --location westeurope \
    --retention-time 30 \
    --workspace-name logic-test
```
```bash
az monitor log-analytics workspace get-shared-keys \
    --resource-group logic-test \
    --workspace-name logic-test \
    --query primarySharedKey \
    --output tsv
```

Please note that this will run the integration immediately and will retry until it is successful.

```bash
az container create \
    --resource-group logic-test \
    --name logic-test \
    --image ubuntu:latest \
    --command '/bin/cat /etc/services' \
    --restart-policy OnFailure \
    --log-analytics-workspace WORKSPACE_ID \
    --log-analytics-workspace-key SHARED_KEY
```

The registration may take some time. Use the show command to check the registration status.
```bash
az provider register \
    --namespace Microsoft.Logic

az provider show \
    --namespace Microsoft.Logic \
    --query registrationState \
    --output tsv
```

In order to do this, we need to create an API connection.
```bash
az logic workflow create \
    --resource-group logic-test \
    --name logic-test \
    --location westeurope \
    --definition logic-test.json
```