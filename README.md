# Azure tech training day

![poster](/images/poster.png)

This workshop provides a detailed overview of the most popular services in Azure. After completing it as a lab, you will have experience regarding how to create a relatively complex solution following best practices. Previous knowledge of the platform (or the supervision of a trainer) is adviced to understand the details of the architecture.

## Initial configuration

![Resource groups](/images/blur-bookcase-business-cds-264544.jpg)

Let's configure our credentials and create the basic resources, including a Resource Group.

```bash
PS1=\\n\\n$PS1

export PREFIX=lab$RANDOM

# Optional, if not using the Web Shell
az login

az account show
az account list-locations --output table
az provider list --output table
az provider register --name Microsoft.KeyVault

az group create --name $PREFIX-rg --location westeurope
```

## Storage accounts

![Storage](/images/vinyl-record-playing-164853.jpg)

We will use a Storage Account to create a very simple static website. A newer and more powerful approach to solve the same problem would be using the [Azure Static WebApps](https://azure.microsoft.com/en-us/services/app-service/static/).

```bash
az storage account create \
  --name ${PREFIX}stacc \
  -g $PREFIX-rg \
  --kind StorageV2 \
  --sku Standard_LRS
```

### Static websites

Take this example as a way to familarize yourself with Storage Accounts and Blob Containers. Also, you will use the command line to transfer files between your laptop (or the cloud shell) and the container.

```bash
az storage blob service-properties update \
  --account-name ${PREFIX}stacc \
  --static-website \
  --404-document 404.html \
  --index-document index.html \
  --output table

mkdir web
wget https://pastebin.com/raw/Y1D5zYNR -O web/index.html
wget https://pastebin.com/raw/jxEqVD29 -O web/404.html

az storage blob upload-batch \
  --account-name ${PREFIX}stacc \
  --source web \
  --destination \$web \
  --output table

az storage account show \
  --name ${PREFIX}stacc \
  --query "primaryEndpoints.web" \
  --output tsv
```

## Relational databases

![Databases](/images/UNIVAC-I-BRL61-0977-wikipedia.jpg)

### Servers and databases

We will create a Azure SQL Server containing one database. And we will open direct access to it by creating a Firewall rule. In this case, this is not a recommended pattern, but the best approach (using Endpoints) will be shown later.

```bash
SQL_PASS=MyPassword$RANDOM
echo $SQL_PASS > sql_pass.txt

DB_PREFIX=$PREFIX

az sql server create \
  --resource-group $DB_PREFIX-rg \
  --name $DB_PREFIX-pokemondb-server \
  --admin-user dbadmin \
  --admin-password $SQL_PASS \
  --output table

az sql db create \
  --resource-group $DB_PREFIX-rg \
  --server $DB_PREFIX-pokemondb-server \
  --name $PREFIX-pokemonDB \
  --auto-pause-delay 600 \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 1 \
  --compute-model Serverless \
  --zone-redundant false \
  --tags Owner=$DB_PREFIX Project=pokemon \
  --output table
```

### Firewall rules

```bash
MY_IP=$(curl ifconfig.me)

az sql server firewall-rule create \
  --resource-group $DB_PREFIX-rg \
  --name $PREFIX-pokemondb-server-fw-rule \
  --server $DB_PREFIX-pokemondb-server \
  --start-ip-address $MY_IP \
  --end-ip-address $MY_IP \
  --output table
```

### Interaction

[mssql-cli](https://docs.microsoft.com/sql/tools/mssql-cli) is a very nice tool to interact with SQL Server and other compatible database engines. We will use it to connect to our instance and check everything has been properly setup.

```bash
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -
sudo curl -o /etc/apt/sources.list.d/microsoft.list "https://packages.microsoft.com/config/ubuntu/$(lsb_release -sr)/prod.list"
sudo apt-get update
sudo apt-get install mssql-cli -y
mssql-cli --version
```

The following instructions will list the content of the database and will execute a [data dump](https://pastebin.com/raw/3jkbTTSq) stored in a pastebin page. Basically, we pipe the content of that file throght `mssql-cli` to insert rows into our new database.

```bash
SQL_URL=$DB_PREFIX-pokemondb-server.database.windows.net

mssql-cli \
  --username dbadmin \
  --password $SQL_PASS \
  --server $SQL_URL \
  --database $PREFIX-pokemonDB \
  --query "\ld"

curl https://pastebin.com/raw/3jkbTTSq -s | \
  mssql-cli \
    --username dbadmin \
    --password $SQL_PASS \
    --server $SQL_URL \
    --database $PREFIX-pokemonDB 
```

## Application secrets

![Identities](/images/shield-id-2.jpg)

[Managed Identities](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview) are the best way to provide credentials to our applications running inside VMs or WebApps. We will use them to allow an application to acces [Azure Key Vault](https://azure.microsoft.com/services/key-vault/), the only correct way to store sensitive information like the connection string to our database.

### Keyvault and User Managed Identities

```bash
az keyvault create \
  --resource-group $PREFIX-rg \
  --name $PREFIX-vault \
  --output table

az identity create \
  --name $PREFIX-pokemonapp-msi \
  --resource-group $PREFIX-rg 

PRINCIPAL=$(az identity show \
  --name $PREFIX-pokemonapp-msi \
  --resource-group $PREFIX-rg \
  --query principalId \
  --output tsv) && echo $PRINCIPAL

az keyvault set-policy \
  --resource-group $PREFIX-rg \
  --name $PREFIX-vault \
  --secret-permission get \
  --object-id $PRINCIPAL \
  --output table
```

![Vaults](/images/abus-brand-close-up-closed-277670.jpg)

### Secret storage

The connection string contains sensitive information, like the database password. We will save it into the Key Vault created previously.

```bash
SQL_CONN=$(az sql db show-connection-string \
  --client odbc \
  --auth-type SqlPassword \
  --name $PREFIX-pokemonDB \
  --server $DB_PREFIX-pokemondb-server \
  --output tsv | \
  awk '{gsub("<username>", "dbadmin", $0); print}' | \
  awk '{gsub("<password>", p, $0); print}' p="$SQL_PASS") && echo $SQL_CONN
  
az keyvault secret set \
  --vault-name $PREFIX-vault \
  --name "PokemonDBConn" \
  --value "$SQL_CONN"
```

## Networking infraestructure


### Virtual networks

![Networking](/images/fortress-pexels-photo-532931.jpeg)

```bash
az network vnet create \
  -g $PREFIX-rg \
  --name $PREFIX-vnet \
  --address-prefix 10.0.0.0/16 \
  --subnet-name $PREFIX-subnet-public \
  --subnet-prefix 10.0.0.0/24 
```

### Managed resource connectivity

![Bridge](/images/stargate.jpg)

The correct way to allow accessing the managed database from a virtual machine **is not** opening Firewall ports like we did before. Instead, we will create a [Service Endpoint](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-service-endpoints-overview) to set a network card of database inside our Virtual Network.

```bash
# This will only work with subscription-wide privileges. Don't worry if it is not your case.
az network vnet list-endpoint-services \
  --location westeurope \
  --output table

az network vnet subnet update \
  --resource-group $PREFIX-rg \
  --vnet-name $PREFIX-vnet \
  --name $PREFIX-subnet-public \
  --service-endpoints Microsoft.Sql \
  --output table

# Subnet and Azure SQL may be in different resource groups, so we need the full resource name
SUBNET_ID=$(az network vnet subnet show --resource-group $PREFIX-rg --vnet-name $PREFIX-vnet --name $PREFIX-subnet-public --query id --output tsv) && echo Public subnet ID: $SUBNET_ID

az sql server vnet-rule create \
  --name $PREFIX-pokemondb-server-firewall-rule \
  --resource-group $DB_PREFIX-rg \
  --server $DB_PREFIX-pokemondb-server \
  --subnet $SUBNET_ID 
```

## Virtual machines

![Dogs](/images/dogs.jpg)

* See https://azureprice.net

### Firewall configuration using NSGs

![Dog](/images/dog.jpg)

Network Security Groups are the preferred way to protect traffic inside the Virtual Networks. They can be attached to both VMs and Subnets, and it is perfectly fine to do it both ways. In this case, we will create a security group opening administration ports (port 22, not the best practice) and web traffic (80 and 8080).

```bash
az network nsg create \
  --name $PREFIX-pokemon-nsg \
  --resource-group $PREFIX-rg 

az network nsg rule create \
  --resource-group $PREFIX-rg \
  --name $PREFIX-pokemon-nsg-ssh \
  --nsg-name $PREFIX-pokemon-nsg \
  --priority 220 \
  --access Allow \
  --source-address-prefixes $MY_IP \
  --destination-port-ranges 22 \
  --protocol Tcp \
  --description "Allow ssh administration (bad idea)" 

az network nsg rule create \
  --resource-group $PREFIX-rg \
  --name $PREFIX-pokemon-nsg-web \
  --nsg-name $PREFIX-pokemon-nsg \
  --priority 200 \
  --access Allow \
  --source-address-prefixes Internet \
  --destination-port-ranges 80 8080 \
  --protocol Tcp \
  --description "Allow traffic to pokemon app" 
```
  
### Virtual machine creation

This Virtual Machine will have the Managed Identity attached to it, allowing access to the Key Vault from where an application (installed later) will retrieve the connection string to the database. Also, we will attach the previously created Network Security Group to it, so we will be able to `ssh` inside the VM and also access the server running inside using our browser.

```bash
# Show all resources created: PublicIP, VMNic, vm-Disk and vm
az vm create \
  --name $PREFIX-pokemon-vm \
  --resource-group $PREFIX-rg \
  --image UbuntuLTS  \
  --admin-username $PREFIX \
  --vnet-name $PREFIX-vnet \
  --subnet $PREFIX-subnet-public \
  --authentication-type ssh \
  --assign-identity $PREFIX-pokemonapp-msi \
  --nsg $PREFIX-pokemon-nsg \
  --generate-ssh-keys \
  --size Standard_DS1_v2 \
  --tags Owner=$PREFIX Project=pokemon Name="Pokemon VM" 
```

### Software configuration with extensions

![Building](/images/2-man-on-construction-site-during-daytime-159306.jpg)

Extensions are not recommended anymore for installing software, as it takes time to execute them. But they are still present in many scenarios, so we will take advantage of this mechanism to install a Pokemon web application inside the previosly created server.

The next instructions will generate the script to be run inside the VM.

```bash
cat << EOF > script-pokemon.sh
#!/bin/sh

sudo apt update
sudo apt -y install curl dirmngr apt-transport-https lsb-release ca-certificates
curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
sudo apt -y install nodejs gcc g++ make

git clone https://github.com/ciberado/pokemon-nodejs
cd pokemon-nodejs
git checkout azure-demo
npm install
export PORT=8080
export KEY_VAULT_URI=https://$PREFIX-vault.vault.azure.net
# Important: avoid blocking the extension mechanism by running the app in background
node app.js > app.log &
EOF
```

Now we will use a Storage Account to store the script.

```bash
az storage blob upload \
  --account-name ${PREFIX}stacc \
  --container-name \$web \
  --file ./script-pokemon.sh \
  --name script-pokemon.sh \
  --output table

SCRIPT_URL=$(az storage account show \
  --name ${PREFIX}stacc \
  --query "primaryEndpoints.web" \
  --output tsv)script-pokemon.sh && echo $SCRIPT_URL
```

Now, we will instruct the VM to execute the *run script* Extension, downloading the script and executing it to install and run the Pokemon application.

```bash
az vm extension set \
  --resource-group $PREFIX-rg \
  --vm-name $PREFIX-pokemon-vm \
  --name customScript \
  --publisher Microsoft.Azure.Extensions \
  --settings "{\"fileUris\": [\"$SCRIPT_URL\"],\"commandToExecute\": \"./script-pokemon.sh\"}" 
```

Finally, we can retrieve the IP of the VM and (hopefully) we will be able to access the logs of the server and the application itself.

```bash
VM_IP=$(az vm list-ip-addresses \
  --resource-group $PREFIX-rg \
  --name $PREFIX-pokemon-vm \
  --query [0].virtualMachine.network.publicIpAddresses[0].ipAddress \
  --output tsv) && echo $VM_IP

ssh $PREFIX@$VM_IP \
  sudo cat /var/lib/waagent/custom-script/download/0/pokemon-nodejs/app.log

echo Click here: http://$VM_IP:8080
```

## Asynchronous architectures

![Treebookhouse](/images/book-crossing-tree.jpg)

### Storage account queues

```bash
az storage queue create \
  --account-name ${PREFIX}stacc \
  --name healthbeats \
  --output table
```

### Permission assignment 

![Keys](/images/antique-crumpled-crumpled-paper-dirty-612800.jpg)

```bash
SA_ID=$(az storage account show --name ${PREFIX}stacc -g $PREFIX-rg --query id --output tsv) && echo $SA_ID

PRINCIPAL=$(az identity show --name $PREFIX-pokemonapp-msi --resource-group $PREFIX-rg --query principalId --output tsv) && echo $PRINCIPAL

until az role assignment create \
  --assignee $PRINCIPAL \
  --role 'Owner' \
  --scope $SA_ID
do
  echo "Trying again role assignment."
  sleep 10
done 

until az role assignment create \
  --assignee $PRINCIPAL \
  --role 'Storage Queue Data Contributor' \
  --scope $SA_ID
do
  echo "Trying again role assignment."
  sleep 10
done 
```

### Queue producer creation

![Written order](/images/person-hands-woman-pen-110473.jpg)

```bash
cat << EOF > script-pokemon-healthbeat.sh
#!/bin/sh

sudo apt update
sudo apt -y install curl dirmngr apt-transport-https lsb-release ca-certificates
curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
sudo apt -y install nodejs gcc g++ make

git clone https://github.com/ciberado/pokemon-nodejs
cd pokemon-nodejs
git checkout azure-demo
npm install
export PORT=8080
export AZURE_STORAGE_NAME=${PREFIX}stacc
export KEY_VAULT_URI=https://$PREFIX-vault.vault.azure.net
node app.js > app.log &
EOF

# This time, using custom-data
az vm create \
  --name $PREFIX-pokemon-vm-healthbeat \
  --resource-group $PREFIX-rg \
  --image UbuntuLTS  \
  --admin-username $PREFIX \
  --vnet-name $PREFIX-vnet \
  --subnet $PREFIX-subnet-public \
  --authentication-type ssh \
  --assign-identity $PREFIX-pokemonapp-msi \
  --nsg $PREFIX-pokemon-nsg \
  --generate-ssh-keys \
  --size Standard_DS1_v2 \
  --tags Owner=$PREFIX Project=pokemon Name="Pokemon VM" \
  --custom-data script-pokemon-healthbeat.sh
  
VM_IP_HC=$(az vm list-ip-addresses \
  --resource-group $PREFIX-rg \
  --name $PREFIX-pokemon-vm-healthbeat \
  --query [0].virtualMachine.network.publicIpAddresses[0].ipAddress \
  --output tsv) && echo $VM_IP_HC


ssh $PREFIX@$VM_IP_HC \
  sudo tail /var/log/cloud-init-output.log --follow
ssh $PREFIX@$VM_IP_HC \
  sudo tail /pokemon-nodejs/app.log --follow

echo Click here: http://$VM_IP_HC:8080

```

### Command line queue consumer

![Welding](/images/man-wearing-welding-helmet-welding-metal-near-gray-brick-1474993.jpg)

```bash
az storage message get \
  --account-name ${PREFIX}stacc \
  --queue-name healthbeats \
  --num-messages 32
```

## HA architectures

![Two bridges](/images/bridges.jpg)

### Load balancers, backends, probes and rules

![Lights](/images/splittedlaser.jpg)

```bash
az network public-ip create \
  --resource-group $PREFIX-rg \
  --name $PREFIX-lb-ip \
  --allocation-method Static \
  --sku Standard \
  --output table

az network lb create \
  --resource-group $PREFIX-rg \
  --name $PREFIX-lb \
  --public-ip-address $PREFIX-lb-ip  \
  --frontend-ip-name $PREFIX-lb-frontend-pool \
  --backend-pool-name $PREFIX-backend-pool \
  --sku Standard \
  --tags Owner=$PREFIX Project=pokemon 

az network lb probe create \
  --resource-group $PREFIX-rg \
  --lb-name $PREFIX-lb \
  --name $PREFIX-lb-probe \
  --port 8080 \
  --protocol http \
  --path /health \
  --interval 30

az network lb rule create \
  --resource-group $PREFIX-rg \
  --name $PREFIX-lb-rule \
  --backend-port 8080 \
  --frontend-port 80 \
  --lb-name $PREFIX-lb \
  --protocol Tcp \
  --backend-pool-name $PREFIX-backend-pool \
  --load-distribution Default \
  --probe-name $PREFIX-lb-probe
```

### Firewall rules

![Dog](images/white-short-coated-dog-160846.jpg)

```bash
az network nsg create \
  --name $PREFIX-pokemon-vmss-nsg \
  --resource-group $PREFIX-rg 

az network nsg rule create \
  --resource-group $PREFIX-rg \
  --name $PREFIX-pokemon-vmss-nsg-rule-from-lb \
  --nsg-name $PREFIX-pokemon-vmss-nsg \
  --priority 200 \
  --access Allow \
  --source-address-prefixes AzureLoadBalancer \
  --destination-port-ranges 8080 \
  --protocol Tcp \
  --description "Allow traffic to pokemon app from any L4 LoadBalancer" 

az network nsg rule create \
  --resource-group $PREFIX-rg \
  --name $PREFIX-pokemon-vmss-nsg-rule-from-troubleshooting \
  --nsg-name $PREFIX-pokemon-vmss-nsg \
  --priority 220 \
  --access Allow \
  --source-address-prefixes "*" \
  --destination-port-ranges 8080 22 \
  --protocol Tcp \
  --description "Allow traffic to pokemon app for troubleshooting" 
```

## VM Fleets with VMSS

![Clones](/images/clones.jpg)

```bash
az vmss create \
  --resource-group $PREFIX-rg \
  --name $PREFIX-pokemon-vmss \
  --computer-name-prefix $PREFIX-pokemon-vmss-vm \
  --instance-count 2 \
  --vnet-name $PREFIX-vnet \
  --subnet $PREFIX-subnet-public \
  --public-ip-per-vm  \
  --public-ip-address-allocation static \
  --load-balancer $PREFIX-lb \
  --backend-pool-name $PREFIX-backend-pool \
  --zones 2 \
  --nsg $PREFIX-pokemon-vmss-nsg \
  --image UbuntuLTS \
  --vm-sku Standard_DS1_v2 \
  --upgrade-policy-mode Automatic \
  --admin-username $PREFIX \
  --generate-ssh-keys \
  --assign-identity $PREFIX-pokemonapp-msi \
  --tags Owner=$PREFIX Project=pokemon \
  --custom-data script-pokemon-healthbeat.sh

VMSS_VM0_IP=$(az vmss list-instance-public-ips \
  --resource-group $PREFIX-rg \
  --name $PREFIX-pokemon-vmss \
  --query [0].ipAddress \
  --output tsv) && echo $VMSS_VM0_IP

ssh $PREFIX@$VMSS_VM0_IP \
  sudo tail /var/log/cloud-init-output.log --follow

ssh $PREFIX@$VMSS_VM0_IP tail /pokemon-nodejs/app.log --follow

LB_IP=$(az network public-ip show \
  --resource-group $PREFIX-rg \
  --name $PREFIX-lb-ip \
  --query ipAddress \
  --output TSV) && echo "Click on http://$LB_IP"
```

## Azure apps service

![Happyness](/images/adult-background-casual-941693.jpg)

### Web app provisioning

```bash
az appservice plan create \
  --resource-group $PREFIX-rg \
  --name ${PREFIX}-prod-plan \
  --sku S1 \
  --is-linux

az webapp create \
  --resource-group $PREFIX-rg \
  --name ${PREFIX}-prod \
  --plan ${PREFIX}-prod-plan \
  --deployment-container-image-name nginx

az webapp log config \
  --resource-group $PREFIX-rg \
  --name ${PREFIX}-prod \
  --application-logging true \
  --detailed-error-messages true \
  --docker-container-logging filesystem 
```

### Slots

![Two phones](/images/selective-focus-photography-of-two-smartphones-with-blank-2606516.jpg)

```bash
az webapp deployment slot create \
  --resource-group $PREFIX-rg \
  --name ${PREFIX}-prod \
  --slot secondary

az webapp log config \
  --resource-group $PREFIX-rg \
  --name ${PREFIX}-prod \
  --slot secondary \
  --application-logging true \
  --detailed-error-messages true \
  --docker-container-logging filesystem 
```

### Web app configuration

```bash
SA_CONN_STR=$(az storage account show-connection-string \
  --resource-group $PREFIX-rg \
  --name ${PREFIX}stacc \
  --query connectionString \
  --output tsv)

az webapp config appsettings set \
  --resource-group $PREFIX-rg \
  --name ${PREFIX}-prod \
  --settings AZURE_STORAGE_CONNECTION_STRING="$SA_CONN_STR" 
  
az webapp config appsettings set \
  --resource-group $PREFIX-rg \
  --name ${PREFIX}-prod \
  --slot secondary \
  --slot-settings AZURE_STORAGE_CONNECTION_STRING="$SA_CONN_STR" 
```

### Container deployment

![Whale](/images/whale-2517325_1280.jpg)

```bash
az webapp config container set \
  --resource-group $PREFIX-rg \
  --name ${PREFIX}-prod \
  --docker-custom-image-name ciberado/pokemon-dashboard:0.0.1 \
  --slot secondary 

WEBAPP_PRODUCTION=$(az webapp show \
  --resource-group $PREFIX-rg \
  --name ${PREFIX}-prod \
  --query defaultHostName \
  --output tsv) && echo Click on https://$WEBAPP_PRODUCTION

WEBAPP_SECONDARY=$(az webapp show \
  --resource-group $PREFIX-rg \
  --name ${PREFIX}-prod \
  --slot secondary \
  --query defaultHostName \
  --output tsv) && echo Click on https://$WEBAPP_SECONDARY
```

### Traffic control

![Traffic](/images/light-trails-on-city-street-327345.jpg)

```bash
az webapp traffic-routing set \
  --resource-group $PREFIX-rg \
  --name $PREFIX-prod \
  --distribution secondary=50 

az webapp deployment slot swap \
  --resource-group $PREFIX-rg \
  --name $PREFIX-prod \
  --action swap \
  --slot secondary 
```

## Azure pipelines

![Gear](/images/colorful-toothed-wheels-171198.jpg)

This [is the tutorial](https://github.com/ciberado/azure-devops-workshop) you are looking for.
