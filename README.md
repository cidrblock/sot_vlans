Overview
==============

The intent of this playbook is to demonstrate the concept of "source of truth" as it relates to network devices.  Currently, in our environment, there is a high percentage of the network device configurations that is not store or tracked anywhere but the device itself or in the archive of the device's configuration.

The ultimate goal is to move the source of truth away from the devices into a source code repository.

We see the benefits of this approach being:

  1) Since the metadata is stored as structured data is can be transformed into an OS specific configuration later.
  2) Devices that should have the same configuration, can draw from the same source.
  3) Eventually, event driven automation will trigger playbook runs so devices will automatically be updated when the repository changes.

As an example, we can move the SOT for the VLANs on NX-OS systems to this playbook.  NX-OS was chosen as an example for a couple reasons:

  1) NX-OS does not store it's VLANs in the running configuration, so the output of "show vlan" needs to be parsed.
  2) This relies on the [ntc_show_command](https://github.com/networktocode/ntc-ansible) module.  In the background it uses [textfsm](http://jedelman.com/home/programmatic-access-to-cli-devices-with-textfsm/) to parse the output of a 'show' command and return structured data.

The playbook performs the following steps:

  1) Determines the OS of the device, since we don't know at run time, all devices are treated as ios.  
  2) Collects and updates each devices host_vars file with the current VLANs if they didn't originally exist.
  3) Again, collects the VLANs on the device to compare with what Ansible knows to be the desired list.
  4) Removes and add vlans as necessary
  5) Reports all changes at the end of the playbook.

The end state is Ansible is now the source of truth for each device's VLAN list.  An modification to the device's host_vars file will result in changes on the device for subsequent runs.

Example and usage:
==============

Update the inventory.txt file with the devices to administer.

We'll run Ansible inside of our [docker](https://github.com/cidrblock/docker_ansible) container since the ntc_show module is installed.

```
docker run --rm -it -v `pwd`:/working ansible-dev
```

Inside the container we need to switch to the working directory and set the username and password via environment variables.

```
cd /working
export ANSIBLE_NET_USERNAME=username
export ANSIBLE_NET_PASSWORD='password'
```

Ensure the host list is what you expect by running --list-hosts for the playbook:

```
➜  /working ansible-playbook -i inventory.txt site.yml --list-hosts

playbook: site.yml

  play #1 (all): all	TAGS: []
    pattern: [u'all']
    hosts (1):
      router.company.net

  play #2 (all): all	TAGS: []
    pattern: [u'all']
    hosts (1):
      router.company.net
```

and the correct tasks will be run:

```
➜  /working ansible-playbook -i inventory.txt site.yml --list-tasks

playbook: site.yml

  play #1 (all): all	TAGS: []
    tasks:
      determine_os : Run show version	TAGS: []
      determine_os : Set fact for os and ssh_accessible	TAGS: []
      retrieve_running : Include OS files for config retrieval	TAGS: []
      retrieve_running : Set the running configuration as a fact	TAGS: []
      save_vlans : Parse the output from 'show vlan'	TAGS: []
      save_vlans : Set a fact for the exisitng VLANs	TAGS: []
      save_vlans : Ensure we have a host specific vars file	TAGS: []
      save_vlans : Load the host's vars so we can add the VLANs to it.	TAGS: []
      save_vlans : Add the existing VLAN list to the contents of the host_vars file	TAGS: []
      save_vlans : Write the host's vars file back	TAGS: []
      save_vlans : Set a fact for vlans as if they were loaded from the inventory	TAGS: []

  play #2 (all): all	TAGS: []
    tasks:
      modify_vlans : Parse the output from 'show vlan'	TAGS: []
      modify_vlans : Set the add and remove VLANs as facts	TAGS: []
      modify_vlans : Build a list of VLAN ids that were added so they are not removed	TAGS: []
      modify_vlans : Add each VLAN	TAGS: []
      modify_vlans : Remove the unneeded VLANs	TAGS: []
      report : Show config changes for device	TAGS: [report]
```

Give it a run in check mode.

Note: Even in check mode this will update your host_vars/xxxx files with the existing VLANs on each device.

```
➜  /working ansible-playbook -i inventory.txt site.yml --check
```

First run
=======
```
➜  /working ansible-playbook -i inventory.txt site.yml --check

PLAY [all] *********************************************************************

TASK [determine_os : Run show version] *****************************************
ok: [router.company.net]

TASK [determine_os : Set fact for os and ssh_accessible] ***********************
ok: [router.company.net]

TASK [save_vlans : Parse the output from 'show vlan'] **************************
ok: [router.company.net]

TASK [save_vlans : Set a fact for the exisitng VLANs] **************************
ok: [router.company.net]

TASK [save_vlans : Ensure we have a host specific vars file] *******************
changed: [router.company.net]

TASK [save_vlans : Load the host's vars so we can add the VLANs to it.] ********
ok: [router.company.net]

TASK [save_vlans : Add the existing VLAN list to the contents of the host_vars file] ***
ok: [router.company.net]

TASK [save_vlans : Write the host's vars file back] ****************************
changed: [router.company.net]

TASK [save_vlans : Set a fact for vlans as if they were loaded from the inventory] ***
ok: [router.company.net]

PLAY [all] *********************************************************************

TASK [modify_vlans : Parse the output from 'show vlan'] ************************
ok: [router.company.net]

TASK [modify_vlans : Set the add and remove VLANs as facts] ********************
ok: [router.company.net]

TASK [modify_vlans : Build a list of VLAN ids that were added so they are not removed] ***
ok: [router.company.net]

TASK [modify_vlans : Add each VLAN] ********************************************

TASK [modify_vlans : Remove the unneeded VLANs] ********************************

TASK [report : Show config changes for device] *********************************
skipping: [router.company.net]

PLAY RECAP *********************************************************************
router.company.net : ok=12   changed=2    unreachable=0    failed=0

➜  /working
```

The device's host_vars file was created if it didn't exists and the current list of VLANs on the device added to it.

Now, modify the VLAN list in the host_vars file and rerun to see the proposed changes.  One VLAN has been modified, one deleted and one added:

```
➜  /working ansible-playbook -i inventory.txt site.yml --check

PLAY [all] *********************************************************************

TASK [determine_os : Run show version] *****************************************
ok: [router.company.net]

TASK [determine_os : Set fact for os and ssh_accessible] ***********************
ok: [router.company.net]

TASK [save_vlans : Parse the output from 'show vlan'] **************************
skipping: [router.company.net]

TASK [save_vlans : Set a fact for the exisitng VLANs] **************************
skipping: [router.company.net]

TASK [save_vlans : Ensure we have a host specific vars file] *******************
skipping: [router.company.net]

TASK [save_vlans : Load the host's vars so we can add the VLANs to it.] ********
skipping: [router.company.net]

TASK [save_vlans : Add the existing VLAN list to the contents of the host_vars file] ***
skipping: [router.company.net]

TASK [save_vlans : Write the host's vars file back] ****************************
skipping: [router.company.net]

TASK [save_vlans : Set a fact for vlans as if they were loaded from the inventory] ***
skipping: [router.company.net]

PLAY [all] *********************************************************************

TASK [modify_vlans : Parse the output from 'show vlan'] ************************
ok: [router.company.net]

TASK [modify_vlans : Set the add and remove VLANs as facts] ********************
ok: [router.company.net]

TASK [modify_vlans : Build a list of VLAN ids that were added so they are not removed] ***
ok: [router.company.net]

TASK [modify_vlans : Add each VLAN] ********************************************
included: /working/roles/modify_vlans/tasks/nxos_vlan_add.yml for router.company.net
included: /working/roles/modify_vlans/tasks/nxos_vlan_add.yml for router.company.net

TASK [modify_vlans : Add/modify vlan 42] ***************************************
changed: [router.company.net]

TASK [modify_vlans : Append changes to log (nxos)] *****************************
ok: [router.company.net]

TASK [modify_vlans : Add/modify vlan 2299] *************************************
changed: [router.company.net]

TASK [modify_vlans : Append changes to log (nxos)] *****************************
ok: [router.company.net]

TASK [modify_vlans : Remove the unneeded VLANs] ********************************
skipping: [router.company.net] => (item={u'name': u'zone2-network-fw-outside', u'vlan_id': u'2299'})
included: /working/roles/modify_vlans/tasks/nxos_vlan_remove.yml for router.company.net

TASK [modify_vlans : Remove vlan 2199] *****************************************
changed: [router.company.net]

TASK [modify_vlans : Append changes to log (nxos)] *****************************
ok: [router.company.net]

TASK [report : Show config changes for device] *********************************
ok: [router.company.net] => {
    "msg": [
        "*** ROLE: modify_vlans ***",
        "vlan 42",
        "name new_vlan",
        "*** ROLE: modify_vlans ***",
        "vlan 2299",
        "name zone2-network-fw-outside-changed",
        "*** ROLE: modify_vlans ***",
        "no vlan 2199"
    ]
}

PLAY RECAP *********************************************************************
router.company.net : ok=15   changed=3    unreachable=0    failed=0
```


Ansible is now the authority for the device's VLAN list.
