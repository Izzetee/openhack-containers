## Run SQL Server

docker network create appnetwork
docker run --name db --network appnetwork -e "ACCEPT_EULA=Y" -e "SA_PASSWORD=blubber123!" -p 1433:1433 -d mcr.microsoft.com/mssql/server:2017-latest

## Create Database

#create database
#update network name, container name and password as needed
#paste each line separately
- docker exec -it db "bash"
-  /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P "blubber123!"
CREATE DATABASE mydrivingDB
SELECT Name from sys.Databases
go


## Load Data
docker run --network appnetwork --name application -e ASPNETCORE_ENVIRONMENT=Local -e SQLFQDN=db -e 'SQLUSER=SA' -e 'SQLPASS=blubber123!' -e SQLDB=mydrivingDB openhack/data-load:v1
 
IMPORTANT: Set the ASPNETCORE_ENVIRONMENT environment variable in POI to Local. This configures the application to skip the use of SSL encryption, allowing connection to the local sql server.

## Build application

Git repo: https://github.com/IzzeTee/openhack-containers
git clone https://github.com/IzzeTee/openhack-containers
cd openhack-containers
 
#build "point of interest" (POI) API image
cd src/poi
docker build -f ../../dockerfiles/Dockerfile_3 -t "tripinsights/poi:1.0" .

## Run application
docker run -d -p 8080:80 --network appnetwork --name poi -e "WEB_SERVER_BASE_URI=http://0.0.0.0" -e "SQL_USER=SA" -e "SQL_PASSWORD=blubber123!" -e "SQL_SERVER=db" -e "ASPNETCORE_ENVIRONMENT=Local" tripinsights/poi:1.0

## Upload images
az login
az account set -s OTA-PRD-252
az acr login --name registrytjp0542AZ

or docker login registrytjp0542.azurecr.io

docker tag tripinsights/{​​​​app}​​​​​​​​​​​:1.0 registrytjp0542.azurecr.io/tripinsights/{​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​app}​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​
docker push registrytjp0542.azurecr.io/tripinsights/{​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​app}

## AKS

REGION_NAME=northeurope
RESOURCE_GROUP=TeamTwoRG
VNET_GROUP=teamResources
SUBNET_NAME=aks-subnet
VNET_NAME=vnet
 
#az network vnet create \
#--resource-group $RESOURCE_GROUP \
#--location $REGION_NAME \
#--name $VNET_NAME \
#--address-prefixes 10.0.0.0/8 \
#--subnet-name $SUBNET_NAME \
#--subnet-prefixes 10.240.0.0/16
 
az network vnet subnet create \
    -g $VNET_GROUP \
    --vnet-name $VNET_NAME \
    -n $SUBNET_NAME \
    --address-prefixes 10.2.1.0/24
 
SUBNET_ID=$(az network vnet subnet show \
    --resource-group $VNET_GROUP \
    --vnet-name $VNET_NAME \
    --name $SUBNET_NAME \
    --query id -o tsv)
 
VERSION=$(az aks get-versions \
    --location $REGION_NAME \
    --query 'orchestrators[?!isPreview] | [-1].orchestratorVersion' \
    --output tsv)
 
AKS_CLUSTER_NAME=awesomecluster
 
az aks create \
--resource-group $RESOURCE_GROUP \
--name $AKS_CLUSTER_NAME \
--vm-set-type VirtualMachineScaleSets \
--node-count 2 \
--load-balancer-sku standard \
--location $REGION_NAME \
--kubernetes-version $VERSION \
--network-plugin azure \
--vnet-subnet-id $SUBNET_ID \
--docker-bridge-address 172.17.0.1/16 \
--generate-ssh-keys \
--attach-acr registrytjp0542 \
--enable-aad \
--enable-rbac
 
 
az aks get-credentials \
    --resource-group $RESOURCE_GROUP \
    --name $AKS_CLUSTER_NAME
 
kubectl get nodes
 
#az aks update -n $AKS_CLUSTER_NAME -g $RESOURCE_GROUP --attach-acr registrytjp0542
 
https://docs.microsoft.com/en-us/azure/aks/azure-ad-rbac?toc=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fazure%2Faks%2Ftoc.json&bc=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fazure%2Fbread%2Ftoc.json
 
 
 
test endpoint: 
kubectl exec -it poi-deployment-778787f96c-lnq7x -- curl -i -X GET http://localhost/api/poi
kubectl exec -it poi-deployment-778787f96c-lnq7x -- curl -i -X GET "http://10.2.0.34/api/user/healthcheck"

kubectl exec -it poi-deployment-778787f96c-lnq7x -- curl -i -X GET "http://userprofile-service.default.svc.cluster.local/api/user"​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​​