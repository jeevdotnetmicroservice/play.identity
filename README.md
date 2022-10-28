# Play.Identity
Play Economy Play.Identity microservice.

## Create and publish package
```powershell
$version="1.0.14"
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
```powershell
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

## Create the signing certificate
```powershell
kubectl apply -f .\kubernetes\signing-cer.yaml -n $namespace
```

## Install the Helm Chart
```powershell
$helmUser=[guid]::Empty.Guid
$helmPassword = az acr login --name $rpname --expose-token --output tsv --query accessToken

$env:HELM_EXPERIMENTAL_OCI=1
helm registry login "$rpname.azurecr.io" --username $helmUser --password $helmPassword

$chartVersion="0.1.0"
helm upgrade identity-service oci://$rpname.azurecr.io/helm/microservice --version $chartVersion -f .\helm\values.yaml -n $namespace --install
```

## Required repository secrets for GitHub workflow
GH_PAT: Created in GitHub user profile --> Settings --> Developer settings --> Personal access token
AZURE_CLIENT_ID: From AAD App Registration
AZURE_SUBSCRIPTION_ID: From Azure Portal subscription
AZURE_TENAN_ID: From Azure AAD properties page

