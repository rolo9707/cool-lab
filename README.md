# The Red Hat Nordics SA lab
Documentation to Red Hat Nordics Lab

## Communication
See: https://github.com/mglantz/cool-lab/blob/main/communication.md

## Development

It is agreed we don't manually do anything. Of course you can play around
freely with demos and such, but anything persistent should be written
as code. We use
[github kan-ban table](https://github.com/orgs/RedHatNordicsSA/projects/1)
to track issues. Definiton of done is that:

* Implementation is automated
* Anyone can run automation based on instructions here.

## Private matters

Secrets like keys, passwords etc. sensitive stuff get's stored into the
[private repo](https://github.com/RedHatNordicsSA/private-lab/).
Ask an admin for the vault key.

## Ansible preparations

Many of the playbooks here create stuff from the scratch. So we ansible will
ask for host key verification. We don't want to stop there, so you can
either change the config file, or just set the environment variable for
time of first time creation of host:

```
[defaults]
host_key_checking = False
```

or

```
export ANSIBLE_HOST_KEY_CHECKING=False
```


We use ansible roles and collections. When starting work from scratch you
need to download dependencies. To get collections and roles to your
workstation, do:

```
ansible-galaxy role install -r roles/requirements.yml -p roles
ansible-galaxy collection install -r collections/requirements.txt -p collections
```

This will download you the requirements.


## Building of RHEL base image with Packer

We start with chicken and egg problem. We can't start a RHEL VM in VMware
until we have RHEL template image in VMware. To solve this, we use
Ansible playbook to command Packer to a) upload RHEL install CD along
with customized kickstart CD to VMware and b) command VMware to build
and store a new VM template out of the incredients.

To build a new template one needs to do:

1. Download RHEL install DVD
2. Put the value into
   [build-rhel-template-packer-vmware.yml](build-rhel-template-packer-vmware.yml)
  ```
  iso:
    rhel_8_5:
      url: "file:///home/itengval/VirtualMachines/rhel-8.5-x86_64-dvd.iso"
      checksum: sha256:1f78e705cd1d8897a05afa060f77d81ed81ac141c2465d4763c0382aa96cadd0
  ```
3. Run the playbook:
  ```
  ansible-playbook -i localhost, -c local -e do_cleanup=false build-rhel-template-packer-vmware.yml
  ```
4. Wait for it....

Ten points for someone who figures out how to avoid the upload. There should
be a way to make packer find it from the VMware datastore directly, but I
haven't figured it out yet. Here are the official
[packer instructions](https://www.packer.io/docs/builders/vsphere/vsphere-iso).

Image can be built with Red Hat image builder, but there is a catch. Red Hat
only supports VMware if template is built on VMware. So it makes sense to do
it this way. For other purposes the image builder is good.

The newly found image will be stored in storage defined in the playbook,
check the vars.

## Ensuring the bastion host

We create the bastion host using the playbook
[setup-bastion.yml](setup-bastion.yml). It creates it from the rhel template,
and does basic configs. The other playbook removes it, or let's you control the
power state.

Create:

```
ansible-playbook  -u root -e "subs_username=$subs_username subs_pw=$subs_pw" setup-bastion.yml
```

## Create / Delete / power off / power on VM

There is [generic playbook](ensure-vm-state.yml) to create VMs from given
template. If you want RHEL you run this:

```
ansible-playbook -i localhost, -c local -e name=rh-test-net ensure-vm-state.yml
```

For power state commands:

```
ansible-playbook -i localhost, -c local -e state=poweredoff -e name=rh-test-net ensure-vm-state.yml
```

And to delete it nicely, unregistering from all places like subs and idm:

```
ansible-playbook  -u root -e "short_name=rh-test-01" -l rh-idm-01.cool.lab  nuke-vm.yml 
```

And bluntly just delete VM, leave subscriptions, insights and idm think it still exists:

```
ansible-playbook -i localhost, -c local -e state=absent -e name=rh-test-net ensure-vm-state.yml
```

There are different values in vars, check them out. Like mem, cpu, network etc tunings.

## Setup IdM hosts

We create one IdM server and a replica. They will be name rh-idm-01.coollab and
rh-idm-02.coollab.

```
ansible-playbook  -u root -e "subs_username=$subs_username subs_pw=$subs_pw" setup-idm.yml
```

## Setup IdM clients

To sign a host into IdM, one needs to add the client to [inventory](hosts) into group called
ipaclients. After inventory is ok, run:

```
ansible-playbook -i hosts -u root -l rh-test-01.cool.lab -e short_name=rh-test-01 setup-idmclient.yml
```

This will ensure the VM hosts are all in IdM.

## Populate Users and Groups to IdM

To add/remove users, update the file
[idm_provision_users.yml](idm_provision_users.yml)
and run:

```
 ansible-playbook -i hosts -u root idm_provision_users.yml
```
