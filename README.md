# Play.Identity
Play Economy Play.Identity microservice.

## Create and publish package
```powershell
$version="1.0.9"
$packageversion="1.0.4"
$owner="jeevdotnetmicroservice"
$gh_pat="[PAT HERE]"

dotnet pack src\Play.Identity.Contracts\ --configuration Release -p:PackageVersion=$packageversion -p:RepositoryUrl=https://github.com/$owner/Play.Identity -o ..\packages

dotnet nuget push ..\packages\Play.Identity.Contracts.$packageversion.nupkg --api-key $gh_pat --source "github"
```

## Build the docker image
```powershell
$rpname="jeevplayeconomy"
$env:GH_OWNER="jeevdotnetmicroservice"
$env:GH_PAT="[PAT HERE]"
docker build --secret id=GH_OWNER --secret id=GH_PAT -t "$rpname.azurecr.io/play.identity:$version" .
```

## Run the docker image
```powershell
$adminPass="[PASSWORD HERE]"
$cosmosDbConnString="[CONN STRING HERE]"
$serviceBusConnString="[CONN STRING HERE]"

docker run -it --rm -p 5002:5002 --name identity -e MongoDbSettings__ConnectionString=$cosmosDbConnString -e ServiceBusSettings__ConnectionString=$serviceBusConnString -e ServiceSettings__MessageBroker="SERVICEBUS" -e IdentitySettings__AdminUserPassword=$adminPass play.identity:$version
```

## Publishing the Docker image
```powershell
az acr login --name $rpname
docker push "$rpname.azurecr.io/play.identity:$version"
```

## Create the kubernetes namespace
```powershell
$namespace="identity"
kubectl create namespace $namespace
```
## Create the Kubernetes pod
```poswershell
kubectl apply -f .\kubernetes\identity.yml -n $namespace
```

### Creating the pod managed identity
```powershell
$rgname="playeconomy"
az identity create --resource-group $rgname --name $namespace
$IDENTITY_RESOURCE_ID=az identity show -g $rgname -n $namespace --query id -otsv

az aks pod-identity add --resource-group $rgname --cluster-name $rpname --namespace $namespace --name $namespace --identity-resource-id $IDENTITY_RESOURCE_ID
```

## Granting access to Key Vault secrets
```powershell
$IDENTITY_CLIENT_ID=az identity show -g $rgname -n $namespace --query clientId -otsv
az keyvault set-policy -n $rpname --secret-permissions get list --spn $IDENTITY_CLIENT_ID
```
