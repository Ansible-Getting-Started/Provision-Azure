# Provision-Azure

For simplicity, we will use Azure’s built-in Cloud Shell command line tool.  Select the Bash option (vs. PowerShell).

Follow the instructions below:

<h2>Setup</h2>

<h3>Make Sure Ansible is Installed</h3>

To check for Ansible, type in:
`which ansible`

Azure CLI will need to be version 2.0.4 or later.  Run the `ansible --version` command to find the version.

<h3>Acquire Azure Credentials and Configure Ansible to Use Them</h3>

For a development environment, create a credentials file for Ansible on your Cloud Shell as follows.

First, type this command:
`az ad sp create-for-rbac`

This creates a service principal and configure its access to Azure resources with a default role assignment.

To find out what your subscription ID is, type in:
`az account show --query "{ subscription_id: id }"`

Output like this should show up; copy this information into a text file so that you can copy/paste it later:

```
{
  "subscription_id": "854c5e9a-ed49-687e-bc7a-96ed7315095"
}
```

Then, type this command in:
`az ad sp create-for-rbac --query '{"client_id": appId, "secret": password, "tenant": tenant}'`

Output like this should show up; copy this information into a text file so that you can copy/paste it later:
```
{
  "client_id": "eec5624a-90f8-4386-8a87-02730b5410d5",
  "secret": "531dcffa-3aff-4488-99bb-4816c395ea3f",
  "tenant": "72f988bf-86f1-41af-91ab-2d7cd011db47"
}
```

Now you’re ready to create a credentials file for Ansible on Cloud Shell.  Input the following commands to change into the `.azure` directory, and create/edit the credentials file:

`cd ~/.azure`

`vi ~/.azure/credentials`

The credentials file itself combines the subscription ID with the output of creating a service principal.  Type `i` in order to insert the following lines into the credentials file - replacing the placeholders with the information from the output you got earlier:

```
[default]
subscription_id=<your-subscription_id>
client_id=<security-principal-appid>
secret=<security-principal-password>
tenant=<security-principal-tenant>
```

Exit insert mode by selecting the Esc key, then save and close the file with `:wq`.

<h3>Verify the Configuration</h3>

In order to make sure everything is configured correctly, let’s use Ansible to create a resource group.

In Cloud Shell, create a file named rg.yml:
`vi rg.yml`

Enter insert mode by selecting the `i` key.

Paste the following code into the editor, keeping in mind that the name variable underneath `azure_rm_resourcegroup` can be anything you want:

```
---
- hosts: localhost
  connection: local
  tasks:
    - name: Create resource group
      azure_rm_resourcegroup:
        name: [esource_group_name]
        location: eastus
      register: rg
    - debug:
        var: rg
```

Exit insert mode by selecting the Esc key.  Save the file and exit the vi editor by entering the `:wq` command.

Run the playbook rg.yml:
`ansible-playbook rg.yml`

The results of running the ansible command should look similar to the following output:
 
```
PLAY [localhost] *********************************************************************************

TASK [Gathering Facts] ***************************************************************************
ok: [localhost]

TASK [Create resource group] *********************************************************************
changed: [localhost]

TASK [debug] *************************************************************************************
ok: [localhost] => {
    "rg": {
        "changed": true,
        "contains_resources": false,
        "failed": false,
        "state": {
            "id": "/subscriptions/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX/resourceGroups/ansible-rg",
            "location": "eastus",
            "name": "config-test",
            "provisioning_state": "Succeeded",
            "tags": null
        }
    }
}

PLAY RECAP ***************************************************************************************
localhost                  : ok=3    changed=1    unreachable=0    failed=0
```

Navigate to the Resource Groups tab on the left side of the Azure user interface to see your newly created resource group!



<h2>Create a Complete VM Environment</h2>

Let's create a playbook that deploys a VM into complete Azure environment.  

First, make sure to create an SSH key pair with the `ssh-keygen` command.  

Then, enter the following:
`cat ~/.ssh/id_rsa.pub`

Copy the resulting output into a text file so that you can paste it into the `ssh_public_keys` portion of `azure_create_vm.yml`.

Next, create a resource group with `az group create` (or stick to the previously-created resource group). The following example creates a resource group named `webinar-test` (can be any name) in the `eastus` location:

`az group create --name webinar-test --location eastus`

Create an Ansible playbook named `azure_create_vm.yml` and paste in the following contents.  The fields in red might be different, according to what you input in the previous playbooks.  Again, remember to enter your own complete public key data in the `key_data` portion:

```
---
- name: Create Azure VM
  hosts: localhost
  connection: local
  tasks:
  - name: Create virtual network
    azure_rm_virtualnetwork:
      resource_group: webinar-test
      name: webinarVnet
      address_prefixes: "10.0.0.0/16"
  - name: Add subnet
    azure_rm_subnet:
      resource_group: webinar-test
      name: webinarSubnet
      address_prefix: "10.0.1.0/24"
      virtual_network: webinarVnet
  - name: Create public IP address
    azure_rm_publicipaddress:
      resource_group: webinar-test
      allocation_method: Static
      name: myPublicIP
  - name: Create Network Security Group that allows SSH
    azure_rm_securitygroup:
      resource_group: webinar-test
      name: webinarNetworkSecurityGroup
      rules:
        - name: SSH
          protocol: Tcp
          destination_port_range: 22
          access: Allow
          priority: 1001
          direction: Inbound
  - name: Create virtual network interface card
    azure_rm_networkinterface:
      resource_group: webinar-test
      name: myNIC
      virtual_network: webinarVnet
      subnet: webinarSubnet
      public_ip_name: myPublicIP
      security_group: webinarNetworkSecurityGroup
  - name: Create VM
    azure_rm_virtualmachine:
      resource_group: webinar-test
      name: WebinarNetworkVM
      vm_size: Standard_DS1_v2
      admin_username: azureuser
      ssh_password_enabled: false
      ssh_public_keys: 
        - path: /home/azureuser/.ssh/authorized_keys
          key_data: "ssh-rsa AAAAB3Nz{snip}hwhqT9h"
      network_interfaces: myNIC
      image:
        offer: CentOS
        publisher: OpenLogic
        sku: '7.3'
        version: latest
```

<h3>How the azure_create_vm.yml Playbook Works</h3>

The following section in an Ansible playbook creates a virtual network named `webinarVnet` in the `10.0.0.0/16` address space:

```
- name: Create virtual network
  azure_rm_virtualnetwork:
    resource_group: webinar-test
    name: webinarVnet
    address_prefixes: "10.0.0.0/16"
```

To add a subnet, the following section creates a subnet named `webinarSubnet` in the `webinarVnet` virtual network:

```
- name: Add subnet
  azure_rm_subnet:
    resource_group: webinar-test
    name: webinarSubnet
    address_prefix: "10.0.1.0/24"
    virtual_network: webinarVnet
```

To access resources across the internet, create and assign a public IP address to your VM. The following section in `azure_create_vm.yml` creates a public IP address named `myPublicIP`:

```
- name: Create public IP address
  azure_rm_publicipaddress:
    resource_group: webinar-test
    allocation_method: Static
    name: myPublicIP
```

Network Security Groups control the flow of network traffic in and out of your VM. The following section in `azure_create_vm.yml` creates a network security group named `webinarNetworkSecurityGroup` and defines a rule to allow SSH traffic on TCP port 22:

```
- name: Create Network Security Group that allows SSH
  azure_rm_securitygroup:
    resource_group: webinar-test
    name: webinarNetworkSecurityGroup
    rules:
      - name: SSH
        protocol: Tcp
        destination_port_range: 22
        access: Allow
        priority: 1001
        direction: Inbound
```

A virtual network interface card (NIC) connects your VM to a given virtual network, public IP address, and network security group. The following section in `azure_create_vm.yml` creates a virtual NIC named myNIC connected to the virtual networking resources you have created:

```
- name: Create virtual network interface card
  azure_rm_networkinterface:
    resource_group: webinar-test
    name: myNIC
    virtual_network: webinarVnet
    subnet: webinarSubnet
    public_ip_name: myPublicIP
    security_group: webinarNetworkSecurityGroup
```

The final step is to create a VM and use all the resources created. The following section creates a VM named `WebinarNetworkVM` and attaches the virtual NIC named myNIC, with your own complete public key data in the `key_data` portion:

```
- name: Create VM
  azure_rm_virtualmachine:
    resource_group: webinar-test
    name: WebinarNetworkVM
    vm_size: Standard_DS1_v2
    admin_username: azureuser
    ssh_password_enabled: false
    ssh_public_keys: 
      - path: /home/azureuser/.ssh/authorized_keys
        key_data: "ssh-rsa AAAAB3Nz{snip}hwhqT9h"
    network_interfaces: myNIC
    image:
      offer: CentOS
      publisher: OpenLogic
      sku: '7.3'
      version: latest
```

<h3>Running the Azure VM Playbook</h3>

To create the complete VM + environment with Ansible, run the playbook as follows:

`ansible-playbook azure_create_vm.yml`

The output looks similar to the following example that shows the VM has been successfully created:

```
PLAY [Create Azure VM] ****************************************************

TASK [Gathering Facts] ****************************************************
ok: [localhost]

TASK [Create virtual network] *********************************************
changed: [localhost]

TASK [Add subnet] *********************************************************
changed: [localhost]

TASK [Create public IP address] *******************************************
changed: [localhost]

TASK [Create Network Security Group that allows SSH] **********************
changed: [localhost]

TASK [Create virtual network interface card] *******************************
changed: [localhost]

TASK [Create VM] **********************************************************
changed: [localhost]

PLAY RECAP ****************************************************************
localhost                  : ok=7    changed=6    unreachable=0    failed=0
```
 
Now check your Azure dashboard!


<h2>Use Ansible to Stop an Azure VM</h2>

Create a file named `stop_vm.yml`, and open it in the VI editor, as follows:

`vi stop_vm.yml`


Enter insert mode by selecting the `i` key.

Paste the following sample code into the editor:
```
---
- name: Stop Azure VM
  hosts: localhost
  connection: local

  tasks:
  - name: Stop the virtual machine
    azure_rm_virtualmachine:
      resource_group: webinar-test
      name: WebinarVM
      allocated: no
```

Exit insert mode by selecting the Esc key.

Save the file and exit the vi editor by entering the following command:
`:wq`


<h3>Run the Sample Ansible Playbook</h3>

Enter the following command:
`ansible-playbook stop_vm.yml`


The output looks similar to the following example that shows the virtual machine has been successfully stopped:
```
PLAY [Stop Azure VM] ********************************************************

TASK [Gathering Facts] ******************************************************
ok: [localhost]

TASK [Stop the Virtual Machine] ***************************************
changed: [localhost]

PLAY RECAP ******************************************************************
localhost                  : ok=2    changed=1    unreachable=0    failed=0
```

<h2>Use Ansible to Start an Azure VM</h2>

Create a file named `start_vm.yml`, and open it in the VI editor, as follows:

`vi start_vm.yml`

Enter insert mode by selecting the `i` key.

Paste the following sample code into the editor:
```
---
- name: Start Azure VM
  hosts: localhost
  connection: local

  tasks:
  - name: Start the virtual machine
    azure_rm_virtualmachine:
      resource_group: webinar-test
      name: WebinarVM
```

Exit insert mode by selecting the Esc key.  Save the file and exit the vi editor by entering the `:wq` command.


<h3>Run the Sample Ansible Playbook</h3>

Enter the following command:
`ansible-playbook start_vm.yml`

The output looks similar to the following example that shows the virtual machine has been successfully started:
```
PLAY [Stop Azure VM] ********************************************************

TASK [Gathering Facts] ******************************************************
ok: [localhost]

TASK [Start the Virtual Machine] ********************************************
changed: [localhost]

PLAY RECAP ******************************************************************
localhost                  : ok=2    changed=1    unreachable=0    failed=0
```
