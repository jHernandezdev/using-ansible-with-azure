### Prerequisites

* [Azure subscription](https://azure.microsoft.com/free/?ref=microsoft.com&utm_source=microsoft.com&utm_medium=docs&utm_campaign=visualstudio)
* [Ansible installed for Azure](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/ansible-install-configure?toc=%2Fazure%2Fansible%2Ftoc.json&bc=%2Fazure%2Fbread%2Ftoc.json#install-ansible-on-an-azure-linux-virtual-machine)
* [Azure PowerShell module](https://docs.microsoft.com/en-us/powershell/azure/install-az-ps?view=azps-3.1.0)


## Table Of Contents
* [Create an Azure Service Principal](#Create-Azure-Service-Principal)
  * [Assign a Role to the Service Principal](#Assign-a-Role-to-the-Service-Principal)
* [Create Azure Credentials](#Create-Azure-Credentials)
  * [Option 1: Ansible Credentials File](#Ansible-Credentials-File)
  * [Option 2: Ansible Environment Variables](#Ansible-Environment-Variables)
* [Run an Ansible Playbook](#Run-Ansible-Playbook)
* [Sources](#Sources)

# Create an Azure Service Principal <a name="Create-Azure-Service-Principal"></a>

```powershell
$credentials = New-Object Microsoft.Azure.Commands.ActiveDirectory.PSADPasswordCredential `
-Property @{ StartDate=Get-Date; EndDate=Get-Date -Year 2024; Password='EnterStrongP@ssw0rd12344321'};

$spSplat = @{
    DisplayName = 'ansibleDev'
    PasswordCredential = $credentials
}

$sp = New-AzAdServicePrincipal @spSplat
```

## Assign a Role to the Service Principal <a name="Assign-a-Role-to-the-Service-Principal"></a>

```powershell
$subId = (Get-AzSubscription -SubscriptionName 'NameOfSubscriptionHere').id

$roleAssignmentSplat = @{
    ObjectId = $sp.id
    RoleDefinitionName = 'Contributor'
    Scope = "/subscriptions/$spId"
}

New-AzRoleAssignment @roleAssignmentSplat

```

# Create Azure credentials <a name="Create-Azure-Credentials"></a>

### Gather Required Values

In order to connect to Azure from Ansible you'll need the following values:

* Subscription Id
* Service Principal AppId
* Service Principal Password
* Tenant Id for the Service Principal

```powershell
$subscriptionId = (Get-AzSubscription -SubscriptionName 'NameOfSubscriptionHere').id

$servicePrincipalAppId = (Get-AzADServicePrincipal -DisplayName ansibledev).ApplicationId

$servicePrincipalPassword = 'Value Specified Previously' 

$tenantId = (Get-AzSubscription -SubscriptionName 'NameOfSubscriptionHere').TenantId
```



### Option 1: Use Ansible Credentials File <a name="Ansible-Credentials-File"></a>

`1.` Create a credentials file

```bash
mkdir ~/.azure
vi ~/.azure/credentials
```

`2.` Populate the required Ansible variables. Replace `<Text>` with actual values. 

```
[default]
subscription_id=<subscription_id>
client_id=<security-principal-appid>
secret=<security-principal-password>
tenant=<security-principal-tenant>
```

### Option 2: Use Ansible Environment Variables <a name="Ansible-Environment-Variables"></a>

`1.` Export environment variables on Ansible server. Replace `<Text>` with actual values.

```
export AZURE_SUBSCRIPTION_ID=<subscription_id>
export AZURE_CLIENT_ID=<security-principal-appid>
export AZURE_SECRET=<security-principal-password>
export AZURE_TENANT=<security-principal-tenant>
```

## Run an Ansible Playbook! <a name="Run-Ansible-Playbook"></a>

`1.` Create a playbook file.

  ```
  vi playbook.yaml
  ```

`2.` Paste playbook contents in.

```
---
- hosts: localhost
  connection: local
  tasks:
    - name: Create resource group
      azure_rm_resourcegroup:
        name: ansible-rg
        location: eastus
      register: rg
    - debug:
        var: rg
```

`3.` Run the playbook using ansible-playbook.

```
ansible-playbook playbook.yaml
```

![Alt Text](https://thepracticaldev.s3.amazonaws.com/i/ug3q82y4ori976h29lcp.png)

### Sources <a name="Sources"></a>

[Using Ansible with Azure](https://docs.microsoft.com/en-us/azure/ansible/ansible-overview)

[Quickstart: Install Ansible on Linux virtual machines in Azure](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/ansible-install-configure?toc=%2Fazure%2Fansible%2Ftoc.json&bc=%2Fazure%2Fbread%2Ftoc.json#install-ansible-on-an-azure-linux-virtual-machine)

[Create an Azure service principal with Azure PowerShell](https://docs.microsoft.com/en-us/powershell/azure/create-azure-service-principal-azureps?view=azps-3.1.0)

[New-AzRoleAssignment](https://docs.microsoft.com/en-us/powershell/module/az.resources/new-azroleassignment?view=azps-3.1.0)
