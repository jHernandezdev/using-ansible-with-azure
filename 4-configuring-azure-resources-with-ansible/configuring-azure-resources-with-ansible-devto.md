


# Table Of Contents
* [Introduction](#introduction)
* [Prerequisites](#prerequisites)
* [Connecting to a Windows Host](#connecting-to-a-windows-host)
* [Setting up an IIS Web Server](#Setting-up-an-IIS-Web-Server)
  * [Ansible Windows Modules](#ansible-windows-modules)
  * [Installing IIS with win_feature](#Installing-IIS-with-win_feature)
  * [Setting the Index Page with win_file](#Setting-the-Index-Page-with-win_copy)
  * [Installing dotnetCore Runtime with win_chocolatey](#Installing-dotnetCore-Runtime-with-win_chocolatey)
  * [Create a Logs Directory with win_file](#Create-a-Logs-Directory-with-win_file)
  * [Restart IIS with Handlers](#Restart-IIS-with-handlers)
* [Using Ansible like PowerShell](#using-ansible-like-powershell)
* [Configuring Azure Resources](#configuring-azure-resources)
* [Conclusion](#conclusion)

# Introduction <a name="introduction"></a>

Ansible has expanded beyond the configuration of machines. Ansible can deploy resources to cloud platforms and provision them using various methods. However, configuring machines was and is where Ansible's core functionality is. After all, Ansible is a configuration management tool. In this tutorial you'll walk through configuring an Azure Windows virtual machine. By the end of the tutorial you will have; setup an IIS web machine, deployed an index.html to the web machine, installed the dotnet core run time with chocolatey, created a logs directory on the web machine, and used an idempotent task to handle IIS resets.

# Prerequisites <a name="prerequisites"></a>

In order for you to follow along with this tutorial you'll need the Ansible server connected to Azure. You'll also need several resources deployed to Azure to support a Windows virtual machine hosted in Azure. And before you can connect to the Windows virtual it also needs to be provisioned and WinRM setup. All the steps to accomplish these tasks are laid out in the previous parts of the series.

{% link https://dev.to/joshduffney/connecting-to-azure-with-ansible-22g2 %}

{% link https://dev.to/joshduffney/deploying-resources-to-azure-with-ansible-1pon %}

{% link https://dev.to/cloudskills/provisioning-azure-resources-with-ansible-be2 %}

{% tag azure %}

# Connecting to a Windows Host <a name="connecting-to-a-windows-host"></a>

Ansible uses variables to determine it's configuration. That configuration also determines how Ansible will attempt to connect and authenticate to a remote target. As expected, there is a difference between the connection configuration for a Linux target and a Windows target. Ansible was also designed for Linux configuration management long before Windows and for that reason much of the default configuration favors Linux. There is no cause for alarm as it is easy to change the connection configuration by defining several variables. 

The variables you need to define depends on your WinRM connection type and the authentication option you've chosen. In this tutorial you'll be using WinRM over HTTPS with self-signed certificates and using NTLM for the authentication. The variables that are described here are required for that configuration. If you've chosen a different configuration the variables you need to define will be different.

_Read more about [Connecting to a Windows Host](https://www.ansible.com/blog/connecting-to-a-windows-host) and [Windows Authentication Options](https://docs.ansible.com/ansible/latest/user_guide/windows_winrm.html#authentication-options)._

It is worth mentioning that Ansible has several places you can define variables. Where you place the variables matters a great deal because Ansible has a strict variable precedence order. Within this tutorial you'll be defining the variables within a playbook. To define variables scoped to an Ansible playbook you'll first define a `vars` section. Inside that vars section is where you can place the variables. You define variables by defining the name of the variable followed by a colon and then the value of the variable `varName:varValue`. For the WinRM connection to work you're required to specify the following variables:

* ansible_user
* ansible_password
* ansible_connection
* ansible_winrm_transport
* ansible_winrm_server_cert_validation

Several of these variables are self explanatory. `ansible_user` and `ansible_password` are the user name and password of a local administrator account on the Windows virtual machine. `ansible_connection` is set to ssh by default, however since you are targeting a Windows virtual machine this must be set to WinRM. `ansible_winrm_server_cert_validation` is set to ignore because the certificates were generated locally to the virtual machine and are self-signed.  


```yaml
vars:
  ansible_user: azureuser
  ansible_password: 'P@ssw0rd12344321'
  ansible_connection: winrm
  ansible_winrm_transport: ntlm
  ansible_winrm_server_cert_validation: ignore
```

_Read more about [Ansible's variable precedence](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable)._

# Setting up an IIS Web Server <a name="Setting-up-an-IIS-Web-Server"></a>

The goal of this tutorial is to configure an Azure virtual machine. You can think of configuration as any changes you need to make inside the virtual machine. These are the types of changes Ansible is typically used for, as it is a configuration management tool. Ansible uses a concept of tasks that execute sequentially. These tasks are created by using Ansible modules. Ansible modules are also called module libraries contain idempotent functionally that allow you to make changes to or gather information from different systems. For example the win_feature module allows you to install or remove Windows features from Windows machine targets. Since this tutorial focuses on configuring a Windows virtual machine you'll focus on the Windows modules Ansible has to offer.

### Ansible Windows Modules <a name"ansible-windows-modules"></a>

Ansible's documentation has an index dedicated to Windows modules. Which are prefixed with `win_` at the beginning of the module name. In this tutorial you'll be using several Ansible modules to install and configure an IIS web server. In order to configure the IIS web server you'll use the following Ansible modules:

* [win_feature](https://docs.ansible.com/ansible/latest/modules/win_feature_module.html)
* [win_copy](https://docs.ansible.com/ansible/latest/modules/win_copy_module.html)
* [win_chocolatey](https://docs.ansible.com/ansible/latest/modules/win_chocolatey_module.html)
* [win_file](https://docs.ansible.com/ansible/latest/modules/win_file_module.html)

_See the full list of [Ansible Windows Modules](https://docs.ansible.com/ansible/latest/modules/list_of_windows_modules.html)._

### Installing IIS with win_feature <a name="Installing-IIS-with-win_feature"></a>

Windows Server comes with a long lists of features you can install. These features determine the services and functionality of the server. In order to enable or install the functionality of an IIS web server, you'll have to install the Windows feature `web-server`. Installing this feature installs IIS for hosting web applications. Ansible allows you to do this by using the `win_feature` module. 

win_feature has a single required parameter, the name. The `name` parameter is used to specify the name of the Windows features you are installing. By default the `state` parameter is set to present. Which means to install the feature, opposed to absent which would uninstall it. However, explicit defining the parameter makes the task's intent clearer. A few optional parameters are used; `include_management_tools` and `include_sub_features`. include_management_tools installs the web-server management tools along with the web-server feature and include_sub_features will install any sub features of web-server. 

```yaml
  - name: Install IIS
    win_feature:
        name: web-server
        include_management_tools: yes
        include_sub_features: yes
        state: present
```

### Setting the Index Page with win_copy <a name="Setting-the-Index-Page-with-win_copy"></a>

Next in the playbook you'll update the index.html within the `inetpub\wwwroot` directory. Which will result in a new landing page for the web server. To do this you'll crate the index.html on the Ansible server and copy the file to the Windows virtual machine. 

_index.html_

```html
<!DOCTYPE html>
<html>
<body>
<h1>Using Azure with Ansible: Ansible was here!</h1>
</body>
</html>
```

The `win_copy` Ansible module is used for copying files to remote hosts on Windows. The src parameter is the path of the file on the local Ansible server. The dest parameter is the remote absolute path to where the file is copied to. force is an optional parameter, but allows you to overwrite existing files, if the content is different. 

```yaml
  - name: Copy index.html to wwwroot
    win_copy:
      src: index.html
      dest: C:\inetpub\wwwroot\index.html
      force: yes
```

### Installing dotnetCore Runtime with win_chocolatey <a name="Installing-dotnetCore-Runtime-with-win_chocolatey"></a>

Next, you'll be installing the donetCore runtime on the web server in order to host self-contained dotnetCore applications. In order to accomplish this you'll be using the Ansible `win_chocolatey` module. Chocolatey is a package manager for Windows, which greatly simplifies the process of installing software on Windows machines. 

win_chocolatey requires a `name` parameter which is the name of the chocolatey package being installed. If you are using the public chocolatey feed you can search for packages [here](https://chocolatey.org/packages). However, you can also host them internally as well. The `version` parameter is optional, but allows you to version lock the installation to a version known to work in your environment. `install_args` is used to pass custom arguments to the installer. In this tutorial the install_args are used to exclude framework installs and only install the runtime from the package. `state` can be used to specify if you wish to install or uninstall a particular package. `notify` is part of the hanlders functionality in Ansible. Handlers allow you to run operations on change and skip it if no change is detected. You'll implement a handler later in the playbook.

```yaml
  - name: install net core iis hosting module with no frameworks
    win_chocolatey:
      name: "dotnetcore-windowshosting"
      version: "3.1.0"
      install_args: "OPT_NO_RUNTIME=1 OPT_NO_SHAREDFX=1 OPT_NO_X86=1 OPT_NO_SHARED_CONFIG_CHECK=1"
      state: present
    notify: restart IIS
```

_Learn more about [Simplify Windows Software Packaging and Automation with Chocolatey](https://www.ansible.com/simplify-windows-software) : AnsibleFest Atlanta Session._

### Create a Logs Directory with win_file <a name="Create-a-Logs-Directory-with-win_file"></a>

Another simple, yet important task when setting up the web server is to create a logs directory. Which will be used to store all the applications log files. To accomplish this you'll use the `win_file` Ansible module. It has two parameters path and state. `Path` is the location on the remote target where the directory will be created. `state` is set to directory, which will create a folder on the Windows virtual machine instead of a file. 

```yaml
  - name: Create logging directory 
    win_file:
        path: c:\logs 
        state: directory
```

### Restart IIS with handlers <a name="Restart-IIS-with-handlers"></a>

Handlers in Ansible provide a way for you to run operations only if a change occurs in a task. The win_chocolatey task defined previously requires a restart of IIS, but only if it's the first run of the playbook. All future runs would not require a restart of IIS because the package for dotnetCore would already be installed. Hanlders give you a way to make the restart of IIS only occur when the state of the dotnetCore task changes.

Handlers are defined under a `handlers` section in the Ansible playbook separate from the tasks. Each handler requires a name. This name is created by the notify name given inside a task. In this tutorial the notify name was set to `restart IIS` within the task that used win_chocolatey to install the dotnetcore runtime. After the name you must define the actions to take. These actions give you the ability to define a set of tasks to run after a change is detected. In this example when the notify `restart IIS` is set to changed this handler will use the Ansible module `win_shell` to execute the command `& {iisreset}`. Which is a command line utility to restart IIS. Ultimately, hanlders make your playbooks safer.

```yaml
  handlers:
    - name: restart IIS
      win_shell: '& {iisreset}'
```

# Using Ansible like PowerShell <a name="using-ansible-like-powershell"></a>

Ansible has the concept of hosts or inventory files. These are used to define all the machines Ansible can target when running a playbook. However, the process of generating, maintaining, and using this feature can be heavy for those wanting to use Ansible for smaller automation and orchestration tasks. PowerShell is a fantastic scripting language that can be used for these types of tasks. PowerShell doesn't have a concept of hosts or inventory files. Instead PowerShell allows you to specify the host names of the machines you wish to target at run time. Ansible can be used in this way as well by targeting the `all` group in the playbook and by using the `-i` parameter followed by a `,` after the list of host names you wish to target. 

_Connection vars defined in a playbook._

```yaml
---
- hosts: all
  vars:
    ansible_user: azureuser
    ansible_password: 'P@ssw0rdUseVaultorVarPrompt'
    ansible_connection: winrm
    ansible_winrm_transport: ntlm
    ansible_winrm_server_cert_validation: ignore
```

_Instead of storing the ansible_password in clear text within the playbook use [var_prompt](https://docs.ansible.com/ansible/latest/user_guide/playbooks_prompts.html) or [Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/playbooks_vault.html) to safely store the password. Examples shown here are for simplicity._


```bash
ansible-playbook playbookName.yaml -i 12.93.107.148,
```

_Executing playbooks without host files or inventories._

_Read more about [Using Ansible like PowerShell](http://duffney.io/UsingAnsibleLikePowerShell)._

# Configuring Azure Resources <a name="configuring-azure-resources"></a>

Putting each of the tasks walked through in this tutorial results in the Ansible playbook you see below. The Ansible playbook starts off by defining a hosts section that targets a built-in Ansible group named `all`. Within the hosts section a `vars` section defines several Ansible variables that are used for connecting to remote machines. These are defined so that Ansible can communicate with a Windows target using WinRM and NTLM for authentication.

Next, are all the `tasks`. Tasks are the individual actions Ansible takes to build up the configuration of the machine targeted by the Ansible playbook. It starts off by installing the Windows feature web-server, it's management tools, and all sub features. This sets up IIS for the Windows virtual machine. Once IIS is installed the playbook copies an index.html file to modify the webserver's landing page. After that dotnetCore's runtime is installed using Chocolatey. This allows for dotnetCore applications to be hosted by the web server. The last task creates a logs directory for the applications to log to. At the very end of the Ansible playbook is a handlers section. This handler uses the notify created by the dotnetCore tasks to determine if IIS needs reset or not.



```yaml
---
- hosts: all
  
  vars:
    ansible_user: azureuser
    ansible_password: 'P@ssw0rdUseVaultorVarPrompt'
    ansible_connection: winrm
    ansible_winrm_transport: ntlm
    ansible_winrm_server_cert_validation: ignore

  tasks:
  - name: Install IIS
    win_feature:
        name: web-server
        include_management_tools: yes
        include_sub_features: yes
        state: present

  - name: Copy index.html to wwwroot
    win_copy:
      src: index.html
      dest: C:\inetpub\wwwroot\index.html
      force: yes

  - name: install net core iis hosting module with no frameworks
    win_chocolatey:
      name: "dotnetcore-windowshosting"
      version: "3.1.0"
      install_args: "OPT_NO_RUNTIME=1 OPT_NO_SHAREDFX=1 OPT_NO_X86=1 OPT_NO_SHARED_CONFIG_CHECK=1"
      state: present
    notify: restart IIS

  - name: Create logging directory
    win_file:
        path: c:\logs
        state: directory

  handlers:
    - name: restart IIS
      win_shell: '& {iisreset}'
```

```bash
ansible-playbook configureWindowsVirtualServer.yaml -i 40.121.157.60,
```

_Run time modified, actual run time ~8 minutes._

![Alt Text](https://thepracticaldev.s3.amazonaws.com/i/mlm4kglenkmgbr2p1ic7.gif)

Confirm the Azure Windows virtual machine was configured properly by using curl against the public Ip address. You should see the index.html contents return.

```bash
curl 40.121.157.60
```

![Alt Text](https://thepracticaldev.s3.amazonaws.com/i/obttghrl6yqxmu2gmeyg.png)

# Conclusion <a name="conclusion"></a>

Whether you manage mutable or immutable infrastructure, defining your Infrastructure as Code is a critical step in that process. Infrastructure as Code has several components to it, but step one is documenting the state of your infrastructure. Ansible gives you the ability to capture your infrastructure's state in YAML files and apply that state with idempotent tasks that you can run over and over again without fear of breaking something. 

Once you start to define your infrastructure stack as code, you'll being to realize the effort involved in the task. Get comfortable using other people's code. Leverage the modules Ansible provides. As well as the modules the Ansible community has built. This will save you time, time you can spend adding true value. 

Ansible isn't the only player in the game. Depending on your platform and technology stack there are many tools and technologies that can do what Ansible does. Take the time to understand why Ansible is or isn't the best fit for you and your team. The dichotomy is to not let this question stop you from experimenting with Ansible. You'll have to use it and gain some fundamental knowledge in order to answer the question of "Is this the right tool?".

Be prepare to defend your choice. Everyone has their favorite vendor, favorite tooling, and favorite languages. Chances are you will run into some resistance when introducing the tooling. Seek to understand their concerns and address them with educated answers, useful training, and documentation. 


<!-- get comfortable using someone else's code. saves you time. allows you to add more value. put your ego aside. -->
