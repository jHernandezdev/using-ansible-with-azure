# Lookup Azure Key Vault Secrets with Ansible

## Step 1 - Create an Azure Key Vault

1. Get the Azure Tenant Id

    ```bash
    #AzCLI
    az account show --subscription "MySubscriptionName" --query tenantId --output tsv

    #Azure PowerShell
    (Get-AzSubscription -SubscriptionName "MySubscriptionName").TenantId
    ```

2. Get the Service Principal Object Id

    ```bash
    #AzCLI
    az ad sp show --id <ApplicationID> --query objectId
    ```

    ```powershell
    #Azure PowerShell
    (Get-AzADServicePrincipal -ApplicationId <ApplicationID> ).id
    ```

3. Create a Resource Group with Ansible

    ```yml
    - name: Create resource group
      azure_rm_resourcegroup:
        name: ansible-infra
        location: eastus
    ```

4. Create Azure Key Vault with Ansible

    ```yml
    - name: Create Ansible Key Vault
      azure_rm_keyvault:
      resource_group: ansible-infra
      vault_name: ansibleKeyVaultDEV
      vault_tenant: <tenantID>
      enabled_for_deployment: yes
      sku:
        name: standard
      access_policies:
        - tenant_id: <tenantID>
          object_id: <servicePrincipalObjectId>
          secrets:
            - get
            - list
            - set
    ```

## Step 2 - Create Secrets in Azure Key Vault with Ansible

1. Register Azure Key Vault Info

    ```yml
    - name: Get Key Vault by name
      azure_rm_keyvault_info:
        resource_group: ansible-infra
        name: ansibleKeyVaultDEV
      register: keyvault
    ```

2. Set Key Vault URI Fact

    ```yml
    - name: set KeyVault uri fact
      set_fact: keyvaulturi="{{ keyvault | json_query('keyvaults[0].vault_uri')}}"
    ```

3. Create a Secret with Ansible

    ```yml
    - name: Create a secret
      azure_rm_keyvaultsecret:
        secret_name: adminPassword
        secret_value: "P@ssw0rd0987654321"
        keyvault_uri: "{{ keyvaulturi }}"
    ```

## Step 3 - Lookup Azure Key Vault Secrets with Ansible

### Option 1: Use AzCLI

### Option 2: Use Azure PowerShell

### Option 3: Use azure_keyvault_secret Ansible Lookup Plugin
