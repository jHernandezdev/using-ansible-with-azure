# Deploy a Windows VM to Azure with Ansible

## Step 1 - Create the playbook

```bash
vi deployWindowsAzureVirtualMachine.yaml
```

## Step 2 - Define the Hosts block

```yml
---
- hosts: localhost
  connection: local
```

## Step 3 - Add Tasks to Deploy an Windows VM to Azure

### Add the Tasks block

```yml
  tasks:
```

### Create a Resource Group

```yml
- name: Create resource group
    azure_rm_resourcegroup:
    name: ansible-rg
    location: eastus
```

### Create a Virtual Network

```yml
- name: Create virtual network
    azure_rm_virtualnetwork:
    resource_group: ansible-rg
    name: vNet
    address_prefixes: "10.0.0.0/16"
```

### Add a Subnet to the Virtual Network

```yml
- name: Add subnet
    azure_rm_subnet:
    resource_group: ansible-rg
    name: webSubnet
    address_prefix: "10.0.1.0/24"
    virtual_network: vNet
```

### Create a Public Ip Address

```yml
    - name: Create public IP address
      azure_rm_publicipaddress:
        resource_group: ansible-rg
        allocation_method: Static
        name: webPublicIP
      register: output_ip_address
```

### Output Public Ip Address

```yml
- name: Output public IP
    debug:
    msg: "The public IP is {{ output_ip_address.state.ip_address }}"
```

### Create a Network Security Group

```yml
- name: Create Network Security Group
    azure_rm_securitygroup:
    resource_group: ansible-rg
    name: networkSecurityGroup
    rules:
        - name: 'allow_rdp'
        protocol: Tcp
        destination_port_range: 3389
        access: Allow
        priority: 1001
        direction: Inbound
        - name: 'allow_web_traffic'
        protocol: Tcp
        destination_port_range:
            - 80
            - 443
        access: Allow
        priority: 1002
        direction: Inbound
        - name: 'allow_powershell_remoting'
        protocol: Tcp
        destination_port_range:
            - 5985
            - 5986
        access: Allow
        priority: 1003
        direction: Inbound
```

### Create a Network Interface

```yml
- name: Create a network interface
    azure_rm_networkinterface:
    name: webNic
    resource_group: ansible-rg
    virtual_network: vNet
    subnet_name: webSubnet
    security_group: networkSecurityGroup
    ip_configurations:
        - name: default
        public_ip_address_name: webPublicIP
        primary: True
```

### Create a Virtual Machine

```yml
- name: Create VM
    azure_rm_virtualmachine:
    resource_group: ansible-rg
    name: winWeb01
    vm_size: Standard_DS1_v2
    admin_username: azureuser
    admin_password: "{{ password }}"
    network_interfaces: webNic
    os_type: Windows
    image:
        offer: WindowsServer
        publisher: MicrosoftWindowsServer
        sku: 2019-Datacenter
        version: latest
```

## Step 4 - Use vars_prompt for the Local Administrator Password

```yml
  vars_prompt:
    - name: password
      prompt: "Enter local administrator password"
```

## Step 5 - Run the Ansible Playbook

```bash
ansible-playbook deployWindowsAzureVirtualMachine.yaml
```
