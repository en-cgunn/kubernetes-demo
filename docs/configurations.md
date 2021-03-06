# Passing secrets and configurations to Pods

As container should be immutable you might to pass information to so you can have different configurations and secrets across different deployments (dev, test, staging, production). Typical examples are database connection strings, API keys, certificates, feature flags or configuration files. In this section we will explore what options are available in Kubernetes.

# Table of Contents
- [Passing secrets and configurations to Pods](#passing-secrets-and-configurations-to-pods)
- [Table of Contents](#table-of-contents)
- [Environmental variables](#environmental-variables)
- [ConfigMaps](#configmaps)
- [Secrets](#secrets)
- [Passing Azure credentials to Pods](#passing-azure-credentials-to-pods)
  - [Prepare resource group with tagged resources](#prepare-resource-group-with-tagged-resources)
  - [Prepare AAD identity (Service Principal)](#prepare-aad-identity-service-principal)
  - [Deploy AAD Pod Identity solution](#deploy-aad-pod-identity-solution)
  - [Deploy identity and identity binding](#deploy-identity-and-identity-binding)
- [External options](#external-options)
  - [Azure KeyVault](#azure-keyvault)
  - [External Etcd or Consul](#external-etcd-or-consul)
- [Cleanup](#cleanup)

# Environmental variables
Easiest way to pass configuration data to your Pods is by using environmental variables. There are downsides with that approach, but it is easy to start with.

Deploy Pod with environmental variables.
```
kubectl apply -f podEnv.yaml
```

Check env is visible inside your container.
```
kubectl exec env-pod -- env
```

There are few issues with this approach:
* Configuration is part of Pod definition therefore it can reveal sensitive information. As deployment object should be stored in version control this is not ideal.
* As configuration is bounded to Pod definition we cannot split roles of someone managing configurations while other team deploying Pods. We cannot distinguish between the two for example in RBAC.
* Environmental variables are loaded during container start and changes cannot be reflected without restart.
* Complex constructs are not working well with environmental variables. For example complex configuration files cannot be passed that way. You can pass individual keys and construct configuration file in your container manually, but that is inconviniet.
* Values are stored in Kubernetes system unencrypted so if attacker would gain access to underlaying Etcd, he can see all passwords

# ConfigMaps
We will solve most of previously outlined issues with ConfigMap. 

First let's deploy ConfigMap with configurations. We will include two key value pairs and one multi-line statement (for example ini config file).

```
kubectl apply -f configMap.yaml
```

In our first example we will create Pod and load all keys from our ConfigMap as environmental variables and check results out.
```
kubectl apply -f podCm1.yaml
kubectl exec cm1-pod -- env
```

We see that simple key:value pairs are OK, but complex configuration file is malformed. Perhaps we should just those simple keys and deal with complex one later. With that we can also rename those key. For example you might ConfigMap with some key:value that is used with multiple different Pods. In those Pods you expect different naming convention for keys. Rather than keeping two ConfigMaps with the same values, we can remap names in Pod definition.
```
kubectl apply -f podCm2.yaml
kubectl exec cm2-pod -- env
```

Now what about more complex value that represents something like configuration file? Another method to pass such information to Pod is leverage Volume and map it as file.
```
kubectl apply -f podCm3.yaml
kubectl exec cm3-pod -- ls /etc/config
kubectl exec cm3-pod -- cat /etc/config/configfile.ini
```

Good thing about accessing data via files is that it gets updated without restart (it might take some time for Kubernetes to reflect this change, can be up to one minute). We will now update config map and test it out. While value in env will remain unchanged, value in file will reflect change.
```
kubectl apply -f ConfigMap2.yaml
kubectl exec cm3-pod -- env | grep newnamekey1
kubectl exec cm3-pod -- cat /etc/config/mycfgkey
```

Sometimes it is not convenient to maintain complex configuration files in ConfigMap yaml. You can also use kubectl and point it to specific file or directory and create ConfigMap that way. Note however that this is not declarative way.
```
kubectl create configmap newmap --from-file=./configs/
kubectl describe cm newmap
```

When you need to deploy a lot of ConfigMaps, Deployments, Services and Secrets and would like to have some variables that change for each deployment consider packaging everything up with [Helm](docs/helm.md)

# Secrets
Currenlty secrets behave pretty much the same as ConfigMaps, but there is difference in how it is implemented internally. Secrets are going to be stored in encrypted way in Kubernetes Etcd and there is option being prepared to use Azure KeyVault to protect encryption keys with specialized system (envelope technique). Since Secrets and ConfigMaps are different object you can apply different RBAC to them. For example one user can be responsible for maitaining ConfigMaps, but someone else is managing Secrets.

From operations perspective only major differenc when using Secrets is that they are store in base64 encoding. Note this is not for confidentiality (consider this plain text), rather to prevent some coding issues with some characters.

Let's deploy some secret.
```
kubectl apply -f secret.yaml
```

Note that password is base64 encoded so it actually stands for this:
```
echo -n 'QXp1cmUxMjM0NTY3OA==' | base64 -d
Azure12345678
```

If you create Secret in imperative way from file kubectl will do base64 encoding automatically.
```
echo -n 'Azure87654321' > ./password.txt
kubectl create secret generic secret2 --from-file=./password.txt 
rm ./password.txt
```

We can use Secret in Pod in similar way to ConfigMaps either via env or files.
```
kubectl apply -f podSecret.yaml
kubectl exec sec -- env | grep SECR
```

# Passing Azure credentials to Pods
Sometimes you might need to access Azure resources from Pod for example to read tags of other resources, create records in Azure DNS, reconfigure Traffic Manager priorities or access secrets in Azure Key Vault from your code. Sure you can create AAD Service Principal and pass username and password to Pod via Kubernetes Secret, but you might need even more secure way to do that. With VMs or WebApps there is concept of Managed Service Identity, when you do pass password directly, but leverage local endpoint to get time-restricted token. With AAD Pod Identity you can do this with Pods also.

## Prepare resource group with tagged resources
For our demo we will create Resource Group with two public IPs with tags. We will want to read those tags from our application in Pod later on.

```
az group create -n mygroup -l westeurope
az network public-ip create -g mygroup -n myip1 --tags key1=value1,key2=value1
az network public-ip create -g mygroup -n myip2 --tags key1=value2,key2=value2
```

## Prepare AAD identity (Service Principal)
Let's create new identity in AAD with no password (so service principal cannot be used with simple authnetication, only via token get within Pod) in MC group (that is group that was automatically created for you during AKS deployment) with read rights for our resource group.

```
export resourceGroup=mygroup
export MCgroupName=MC_aksgroup_akscluster_westeurope
export subscription=xxx-xxx-xxx
export principalid=$(az identity create --name tomas-fromkube --resource-group $MCgroupName --query 'principalId' -o tsv)
az role assignment create --role Reader --assignee $principalid --scope /subscriptions/$subscription/resourcegroups/$mygroup
```

## Deploy AAD Pod Identity solution
```
kubectl apply -f https://github.com/Azure/aad-pod-identity/raw/master/deploy/infra/deployment-rbac.yaml
```

## Deploy identity and identity binding
```
kubectl apply -f aadPodIdentityWithBinding.yaml
```



# External options
There might be reasons not to bound any of that with Kubernetes system. Maybe you have very high standards for storing and managing secrets in dedicated hardware-supported (HSM) solutions like Azure Key Vault. Or you have complex configurations you want to centralize and make available not only for apps running in single Kubernetes cluster, but many clusters, Azure Container Instances, VMs or PaaS. In that case you might consider deploying centralized configuration store such as Azure App Configration Service, Etcd or Consul.

## Using AAD Pod Identity to access Azure Key Vault
First configure AAD Pod Identity in your cluster using [this guide](docs/rbac.md#aad-pod-identity-to-access-azure-resources)

Create Azure Key Vault, store some secret and grant myaccount1 access.

```bash
export keyvaultname=tomasvault123
az keyvault create -n $keyvaultname -g aks
az keyvault secret set -n mysecret --vault-name $keyvaultname --value superpassword
az keyvault set-policy -n $keyvaultname \
    --object-id $(az identity show -g aks -n myaccount1 --query principalId -o tsv) \
    --secret-permissions get 
```

Use AAD Pod Identity to get access token for Key Vault and get secret.

```bash
kubectl exec -ti mybox -n app1 -- bash
export token=$(curl -s http://169.254.169.254/metadata/identity/oauth2/token?resource=https://vault.azure.net | jq -r '.access_token')
curl -H "Authorization: Bearer ${token}" https://tomasvault123.vault.azure.net/secrets/mysecret?api-version=7.0
```

## Azure Key Vault FlexVolume
In previous example we have used AAD Pod Identity to get token for Azure Key Vault and access secrets via API. This requires your code to have such support. Sometimes it is not feasible to change your code. That can be solved with Key Vault FlexVolume driver which handles getting secret from Key Vault and mapping to container as file.

Make sure you completed previous lab with AAD Pod Identity and Key Vault. There is one addition permission needed. In prevous lab we access Key Vault secret directly so we needed just permission as part of Key Vault (set-policy). Flex Volume driver also needs Read access to Key Vault ARM resources. Let's add it.

```bash
az role assignment create --role Reader \
    --assignee-object-id $(az identity show -n myaccount1 -g aks --query principalId -o tsv) \
    --scope $(az keyvault show -n $keyvaultname -g aks --query id -o tsv)
```

Install Key Vault FlexVolume driver.

```bash
kubectl create -f https://raw.githubusercontent.com/Azure/kubernetes-keyvault-flexvol/master/deployment/kv-flexvol-installer.yaml
```

```bash
cat > podKeyVaultVolume.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: keyvaulttest
    aadpodidbinding: identity1
  name: keyvaulttest
spec:
  containers:
  - name: keyvaulttest
    image: tkubica/mybox:latest
    volumeMounts:
    - name: test
      mountPath: /kvmnt
      readOnly: true
  volumes:
  - name: test
    flexVolume:
      driver: "azure/kv"
      options:
        usepodidentity: "true"
        keyvaultname: $keyvaultname    
        keyvaultobjectnames: mysecret
        keyvaultobjecttypes: secret
        keyvaultobjectversions: ""
        resourcegroup: aks 
        subscriptionid: $(az account show --query id -o tsv)
        tenantid: $(az account show --query tenantId -o tsv)
EOF
```

Apply Pod and test password file was mapped to container file system.

```bash
kubectl apply -f podKeyVaultVolume.yaml -n app1
kubectl exec keyvaulttest -n app1 -- cat /kvmnt/mysecret
```

## Using Azure App Configuration Service,
Azure Key Vault is universal way to store secrets such as keys and certificates that can be used from any environment and programming language - Kubernetes Pods, virtual machines, Javascript clients or native mobile apps. Nevertheless it is not designed for storing complex configurations and feature flags.

What is alternative to ConfigMaps that could be managed outside of Kubernetes for more universal configuration management of your apps? We will now test Azure App Configuration Service - simple key-valut store and feature flag management as a service.

Create App Configuration (make sure you have latest Azure CLI installed as appconfig is pretty new as time of this writing).

```bash
az appconfig create -n tomasappconfig123 \
    -g aks \
    -l westeurope
az appconfig kv set -y --name tomasappconfig123 \
    --key myconfigkey1 \
    --value myvalue1
```

Store can be accessed using AAD Pod Identity / Managed Service Identity (resources running in Azure), AAD Service Principal or simple key. We will use AAD Pod Identity in this example. We can use various languages SDKs, but for simplicity we will use just azure CLI.

Grant our identity created previously Reader role on our App Configuration resource.

```bash
az role assignment create --role Contributor \
    --assignee-object-id $(az identity show -n myaccount1 -g aks --query principalId -o tsv) \
    --scope $(az appconfig show -g aks -n tomasappconfig123 --query id -o tsv)
```

Connect to mybox in app1 namespace we previously configured for AAD Pod Identity. Authenticate Azure CLI using managed identity and get configuration.

```bash
kubectl exec -ti -n app1 mybox -- bash
az login --identity
az appconfig kv list --name tomasappconfig123 --key myconfigkey1
```

# Cleanup

```
kubectl delete namespace app1
kubectl delete -f podEnv.yaml
kubectl delete -f podCm1.yaml
kubectl delete -f podCm2.yaml
kubectl delete -f podCm3.yaml
kubectl delete -f podSecret.yaml
kubectl delete -f secret.yaml
kubectl delete secret secret2
kubectl delete cm newmap
kubectl delete -f configMap.yaml
```