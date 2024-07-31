# Project Steps to Configure Load Balancer in Azure🚀
## 🌟Here’s a step-by-step guide to achieve your project objectives using the Azure CLI or Azure Cloud Shell on your local machine:🌟
#!/bin/bash

## Variables
RESOURCE_GROUP="EcommercePlatformRG"
LOCATION="eastus"
VNET_NAME="EcommerceVNet"
VNET_PREFIX="10.0.0.0/16"
SUBNET_PREFIX="10.0.1.0/24"
BASTION_SUBNET_PREFIX="10.0.2.0/24"
NAT_GATEWAY_NAME="EcommerceNATGateway"
NAT_GATEWAY_PUBLIC_IP="NATGatewayPublicIP"
BASTION_NAME="EcommerceBastion"
BASTION_PUBLIC_IP="BastionPublicIP"
LB_NAME="EcommerceLB"
LB_FRONTEND_IP="LBPublicIP"
LB_BACKEND_POOL="EcommerceBackendPool"
HEALTH_PROBE_NAME="HTTPProbe"
LB_RULE_NAME="HTTPRule"
VM_SIZE="Standard_DS1_v2"
VM_IMAGE="Win2019Datacenter"
ADMIN_USERNAME="azureuser"
ADMIN_PASSWORD="P@ssw0rd1234"
VM1_NAME="lb-vm1"
VM2_NAME="lb-vm2"
ZONE1="1"
ZONE2="2"

## Step 1: Create Resource Group
az group create --name $RESOURCE_GROUP --location $LOCATION

## Step 2: Create Virtual Network and Subnets
az network vnet create \
  --name $VNET_NAME \
  --resource-group $RESOURCE_GROUP \
  --address-prefix $VNET_PREFIX \
  --subnet-name ResourceSubnet \
  --subnet-prefix $SUBNET_PREFIX

az network vnet subnet create \
  --address-prefix $BASTION_SUBNET_PREFIX \
  --name AzureBastionSubnet \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME

## Create NAT Gateway and Public IP
az network public-ip create \
  --resource-group $RESOURCE_GROUP \
  --name $NAT_GATEWAY_PUBLIC_IP \
  --sku Standard \
  --zone 1

az network nat gateway create \
  --resource-group $RESOURCE_GROUP \
  --name $NAT_GATEWAY_NAME \
  --public-ip-addresses $NAT_GATEWAY_PUBLIC_IP \
  --idle-timeout 10

az network vnet subnet update \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --name ResourceSubnet \
  --nat-gateway $NAT_GATEWAY_NAME

## Step 3: Create Azure Bastion
az network public-ip create \
  --resource-group $RESOURCE_GROUP \
  --name $BASTION_PUBLIC_IP \
  --sku Standard \
  --zone 1

az network bastion create \
  --name $BASTION_NAME \
  --resource-group $RESOURCE_GROUP \
  --vnet-name $VNET_NAME \
  --public-ip-address $BASTION_PUBLIC_IP

## Step 4: Create Load Balancer and Public IP
az network public-ip create \
  --resource-group $RESOURCE_GROUP \
  --name $LB_FRONTEND_IP \
  --sku Standard \
  --zone 1

az network lb create \
  --resource-group $RESOURCE_GROUP \
  --name $LB_NAME \
  --sku Standard \
  --frontend-ip-name $LB_FRONTEND_IP \
  --public-ip-address $LB_FRONTEND_IP

az network lb backend-pool create \
  --resource-group $RESOURCE_GROUP \
  --lb-name $LB_NAME \
  --name $LB_BACKEND_POOL \
  --vnet $VNET_NAME \
  --subnet ResourceSubnet

az network lb probe create \
  --resource-group $RESOURCE_GROUP \
  --lb-name $LB_NAME \
  --name $HEALTH_PROBE_NAME \
  --protocol Http \
  --port 80 \
  --path /

az network lb rule create \
  --resource-group $RESOURCE_GROUP \
  --lb-name $LB_NAME \
  --name $LB_RULE_NAME \
  --protocol Tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name $LB_FRONTEND_IP \
  --backend-pool-name $LB_BACKEND_POOL \
  --probe-name $HEALTH_PROBE_NAME

## Step 4: Create Virtual Machines
az vm create \
  --resource-group $RESOURCE_GROUP \
  --name $VM1_NAME \
  --location $LOCATION \
  --nics $(az network nic create --resource-group $RESOURCE_GROUP --name ${VM1_NAME}Nic --vnet-name $VNET_NAME --subnet ResourceSubnet --query 'NewNIC.id' --output tsv) \
  --image $VM_IMAGE \
  --admin-username $ADMIN_USERNAME \
  --admin-password $ADMIN_PASSWORD \
  --size $VM_SIZE \
  --zone $ZONE1

az vm create \
  --resource-group $RESOURCE_GROUP \
  --name $VM2_NAME \
  --location $LOCATION \
  --nics $(az network nic create --resource-group $RESOURCE_GROUP --name ${VM2_NAME}Nic --vnet-name $VNET_NAME --subnet ResourceSubnet --query 'NewNIC.id' --output tsv) \
  --image $VM_IMAGE \
  --admin-username $ADMIN_USERNAME \
  --admin-password $ADMIN_PASSWORD \
  --size $VM_SIZE \
  --zone $ZONE2

## Step 5: Install IIS on Both VMs
az vm run-command invoke \
  --resource-group $RESOURCE_GROUP \
  --name $VM1_NAME \
  --command-id RunPowerShellScript \
  --scripts "Install-WindowsFeature -name Web-Server -IncludeManagementTools"

az vm run-command invoke \
  --resource-group $RESOURCE_GROUP \
  --name $VM2_NAME \
  --command-id RunPowerShellScript \
  --scripts "Install-WindowsFeature -name Web-Server -IncludeManagementTools"

## Add VMs to Load Balancer Backend Pool
az network nic ip-config address-pool add \
  --address-pool $LB_BACKEND_POOL \
  --ip-config-name ipconfig1 \
  --nic-name ${VM1_NAME}Nic \
  --resource-group $RESOURCE_GROUP \
  --lb-name $LB_NAME

az network nic ip-config address-pool add \
  --address-pool $LB_BACKEND_POOL \
  --ip-config-name ipconfig1 \
  --nic-name ${VM2_NAME}Nic \
  --resource-group $RESOURCE_GROUP \
  --lb-name $LB_NAME

## Output the Load Balancer Public IP for Testing
LB_PUBLIC_IP=$(az network public-ip show --resource-group $RESOURCE_GROUP --name $LB_FRONTEND_IP --query 'ipAddress' --output tsv)
echo "Load Balancer Public IP: $LB_PUBLIC_IP"

