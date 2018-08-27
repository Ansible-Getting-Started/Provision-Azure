# Provision-Azure
Provisioning Azure VMs using playbooks on Cloud Shell.

For simplicity, we will use Azure’s built-in CloudShell command line tool.  Select the Bash option (vs. PowerShell).

Follow the instructions below:

<h1>Make Sure Ansible is Installed</h1>

To check for Ansible, type in:
`which ansible`

Azure CLI will need to be version 2.0.4 or later.  Run the `az --version` command to find the version.

<h1>Acquire Azure Credentials and Configure Ansible to Use Them</h1>

For a development environment, create a credentials file for Ansible on your Cloud Shell as follows.

First, type this command:
`az ad sp create-for-rbac`

This creates a service principal and configure its access to Azure resources with a default role assignment.

To find out what your subscription ID is, type in:
`az account show --query "{ subscription_id: id }"`

Output like this should show up; copy this information into a text file so that you can copy/paste it later:

```{
  "subscription_id": "854c5e9a-ed49-687e-bc7a-96ed7315095"
}```
 
Then, type this command in:
`az ad sp create-for-rbac --query '{"client_id": appId, "secret": password, "tenant": tenant}'`

Output like this should show up; copy this information into a text file so that you can copy/paste it later:
```{
  "client_id": "eec5624a-90f8-4386-8a87-02730b5410d5",
  "secret": "531dcffa-3aff-4488-99bb-4816c395ea3f",
  "tenant": "72f988bf-86f1-41af-91ab-2d7cd011db47"
}```

Now you’re ready to create a credentials file for Ansible on Cloud Shell.  Input the following commands to change into the `.azure` directory, and create/edit the credentials file:

`cd ~/.azure`
`vi ~/.azure/credentials`

The credentials file itself combines the subscription ID with the output of creating a service principal.  Type `i` in order to insert the following lines into the credentials file - replacing the placeholders with the information from the output you got earlier:

```[default]
subscription_id=<your-subscription_id>
client_id=<security-principal-appid>
secret=<security-principal-password>
tenant=<security-principal-tenant>```

Exit insert mode by selecting the Esc key, then save and close the file with `:wq`.

<h1>Verify the Configuration</h1>
In order to make sure everything is configured correctly, let’s use Ansible to create a resource group.
In CloudShell, create a file named rg.yml:
vi rg.yml
Enter insert mode by selecting the ‘i’ key.

Paste the following code into the editor, keeping in mind that the name variable underneath azure_rm_resourcegroup can be anything you want:

```---
- hosts: localhost
  connection: local
  tasks:
    - name: Create resource group
      azure_rm_resourcegroup:
        name: config-test
        location: eastus
      register: rg
    - debug:
        var: rg
```

Exit insert mode by selecting the Esc key.  Save the file and exit the vi editor by entering the `:wq` command.

Run the playbook rg.yml:
`ansible-playbook rg.yml`

The results of running the ansible command should look similar to the following output:
 
```PLAY [localhost] *********************************************************************************

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


