# Azure deployment templates

Templates to facilitate deployment of new signers on Azure.

## Prequisites

This setup assumes you've got the Azure CLI installed and that you've already created the Key Vault with the secret.

The container also requires a Log Analytics workspace id and key. To acquire those, [create a Log Analtyics workspace](https://docs.microsoft.com/en-us/azure/azure-monitor/learn/quick-create-workspace). If you won't be using Log Analytics, you'll need to cut those settings from the `container-template.json` file.

## Choose subscription and resource group

```bash
az account set --subscription {YOUR_SUB}
RESOURCE_GROUP={YOUR_RG}
```

## Deploy the database (postgres)

Fill in the `TODO` placeholder params in `db-parameters.json` and then run the following:

```bash
SERVER_NAME={YOUR_SERVER_NAME}

az deployment group create \
  --resource-group $RESOURCE_GROUP \
  --template-file db-template.json \
  --parameters @db-parameters.json \
  --parameters serverName=${SERVER_NAME}

az postgres db create \
  --name phoneNumberPrivacy \
  --resource-group $RESOURCE_GROUP \
  --server-name $SERVER_NAME

az postgres db list --resource-group $RESOURCE_GROUP --server-name $SERVER_NAME

# Allow access to Azure services
az postgres server firewall-rule create -g $RESOURCE_GROUP -s $SERVER_NAME -n AllowAllWindowsAzureIps --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0
```

You'll also need to run the db migrations, either via a special docker command or by temporarily whitelisting your dev box (see parent [Readme](../README.md)).

## Create a managed identity, add it to container-parameters and grant it access to keyvault

If a managed identity already exists for the signer, you can get it by clicking on the managed identity resource and looking under `properties -> Resource ID`, then just add that string to the container-parameters file. If not, you can create a new managed identity with the following command:

```bash
az identity create \
  --resource-group $RESOURCE_GROUP \
  --name $CONTAINER_NAME
```

The command will output some json, and the string you need to add to the container-parameters file will be under the "id" field. You can then grant the new managed identity access to the keyvault with the following command, or via the azure portal.

```bash
RESOURCE_ID={YOUR_RESOURCE_ID}
az keyvault set-policy \
  --name $KEYVAULT_NAME \
  --resource-group $RESOURCE_GROUP \
  --object-id $RESOURCE_ID \
  --secret-permissions get
```

## Deploy the container instance

Fill in the `TODO` placeholder params in `container-parameters.json`. The `dnsNameLabel` will act as the prefix for your cointainer hostname. Run the following:

```bash
CONTAINER_NAME={YOUR_CONTAINER_NAME}

KEYVAULT_NAME={YOUR_VAULT_NAME}

az deployment group create \
  --resource-group $RESOURCE_GROUP \
  --template-file container-template.json \
  --parameters @container-parameters.json \
  --parameters containerName=$CONTAINER_NAME
```

## Deploy the front door for TLS Termination

```bash
CONTAINER_FQDN=$(az container show \
  --resource-group $RESOURCE_GROUP \
  --name $CONTAINER_NAME \
  --query ipAddress.fqdn --out tsv)

FRONTDOOR_NAME={YOUR_FRONTDOOR_NAME}

az deployment group create \
  --resource-group $RESOURCE_GROUP \
  --template-file frontdoor-template.json \
  --parameters frontdoors_name=$FRONTDOOR_NAME \
  --parameters backend_url=$CONTAINER_FQDN
```

## Monitoring

### Logging

The logs from the container should flow automatically into the configured Log Analytics workspace.

To see errors, execute a query like the following, which you can save for convinient use in creating alerts:
`ContainerInstanceLog_CL | where Message contains "celo_odis_err"`

### Metrics

We use prometheus metrics to generate alerting and dashboards. For the most up to date on-call information please see the Runbook (https://docs.google.com/document/u/1/d/1fSN2_J-OHMr1TqAbj1_i5p7sFgBpZ9xsDIe9WeG4FV8/edit).

### Deploy the prometheus server and sidecar

We use prometheus metrics to collect real time data from signers. The prometheus server scrapes these metrics from the `/metrics` endpoint on the signers and sends them to the sidecar container via a shared volume. The sidecar then reformats the metrics data and exports it to stackdriver.

To deploy the prometheus server and sidecar you must first find the `prometheus-service-account-key` json and paste it into the `prometheus-service-account-key.json` file. Make sure you remember to delete this key from the file again after you finish the deployment and before you push to github! The key can be found by searching for `Service Accounts` in GCP and then creating a service account with `metrics write` permissions if one does not already exist. To get the key, go to the `Actions` tab on the service account and select `Create key`. This should download some JSON to your computer that you can copy and paste into the `prometheus-service-account-key.json` file.

Once you have the service account key, you should fill in the missing fields labeled `"TODO"` in `prometheus-parameters.yaml`. The location parameters should be filled in with the location of the resource group. For the `prometheus.yaml` and `prometheus-service-account-key.json` parameters, you must base64 encode the contents of the `prometheus.yaml` and `prometheus-service-account-key.json` files respectively, and then paste the results into the file. If the `prometheus.yaml`parameter is already filled in and you haven't changed the contents of the `prometheus.yaml` file, then you can leave what is there.

```bash
cat <filename> | base64
```

Make sure to change all these values back to `"TODO"` after you deploy and before committing to master. If you accidentally push the service account key to github, simply delete the service account and create a new one. Then, use the new service account to generate a key and repeat the steps above to redeploy the prometheus server and sidecar.
