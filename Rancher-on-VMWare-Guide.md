# Getting Started With Rancher on VMWare

This guide aims to help people getting started with deploying a Rancher cluster on VMWare using CENTOSStream Templates, hopefully someone finds this useful. This guide is not 100% complete some VMWare knowledge is required but the beefy steps are here.

grtz Alm8 | aleckz999@gmail.com


## PreRequisites

Prequisites to follow this guide:

1. Access to your vcenter as administrator
2. Linux management jumpbox preferably centos

## Deployment Process

The deployment process for Rancher on VMWare is as follows:

1. Prepare a VMWare CENTOSStream Template
2. Prepare vCenter Network Protocol Profiles
3. Prepare Rancher Service Account Credentials in vCenter
4. Deploy a 1 node Rancher Management Cluster
5. Configure Cloud Credentials in Rancher GUI
6. Configure Node Templates in Rancher GUI

## VMWare CENTOSStream Template Preparation

[Download](https://www.centos.org/centos-stream/) the ISO Image in VMWare, the image used in this example is CENTOSStream 8. Upload this ISO into a Datastore and use it to create a new VM. [This guide](https://whmcsglobalservices.com/how-to-create-centos8-vm-template-for-vmware/) will help you if needed

Once the VM is created Once logged in as root, edit the network configuration to put the machine on the network with the below command:

> nmtui

Update the OS packages to the latest versions

> dnf upgrade *

The below command installs the required packages the template to work with Rancher.

> dnf install open-vm-tools perl cloud-init

After performing a software upgrade remove the network adapter files in the network scripts folder

> cd /etc/sysconfig/network-scripts  
> ls -l
> rm ifcfg-ens192

Halt the server

> halt

Convert the VM to a template in your datacenter and give it a logical name.

## Prepare vCenter Network Protocol Profiles

[Follow this guide to deploy the required network protocol profile in your datacenter](https://www.virtualthoughts.co.uk/2020/03/29/rancher-vsphere-network-protocol-profiles-and-static-ip-addresses-for-k8s-nodes/) More info from vmware about Network protocol profiles is [here.](https://docs.vmware.com/en/VMware-vSphere/7.0/com.vmware.vsphere.networking.doc/GUID-D24DBAA0-68BD-49B9-9744-C06AE754972A.html)
In summary you can allocate a pool of IPs in a subnet along with the network IP, Subnet mask, gateway and DNS info. This information can be allocated to requested machines similar to how DHCP works. The Pool can be queried via vmware tools on boot of a vm using a cloud-init script on the VM which is why open-vm-tools perl and cloud-init is required.

## Prepare Rancher Service Account Credentials in vCenter

You can either use an account in active directory if your vcenter is connected to your AD or you can use an account in the vspher.local domain.
This account should be created and given administrator role on all objects in the datacenter where you would like to provision the Rancher Workers and Masters.
See the [VMWare documentation](https://docs.vmware.com/en/VMware-vSphere/7.0/com.vmware.vsphere.security.doc/GUID-5372F580-5C23-4E9C-8A4E-EF1B4DD9033E.html) for this.
It is important that the user account format is serviceaccount@domain.com when using the account in rancher.

## 1 Node Rancher Management Cluster Install

The Rancher Management Cluster is used to create additional kubernetes clusters on the VMWare platform and on other platforms.
This install is automated by ansible or can be done manually using the shell commands.

Create a ansible playbook folder:

> mkdir ~/ansible/centoss-install-rancher-management-host-playbook  

Create the ansible playbook file:

> touch ~/ansible/centoss-install-rancher-management-host-playbook/main.yml  

Edit the file and enter the below yaml:

> vi ~/ansible/centoss-install-rancher-management-host-playbook/main.yml  

```yaml
---
#ansible main.yml 
- name: centoss install rancher management host
  hosts: all
  become: yes
  tasks:
  - name: add centos docker repo
    ansible.builtin.shell:
      cmd: dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
  - name: add docker repo to centos
    ansible.builtin.dnf:
      name: docker-ce
      state: latest
  - name: enable net bridge systemctl settings
    lineinfile:
      path: /etc/sysctl.conf
      line: net.bridge.bridge-nf-call-iptables=1
  - name: Ensure that docker is started
    ansible.builtin.service:
      name: docker
      enabled: true
      state: started
  - name: create docker group
    ansible.builtin.group:
      name: docker
      state: present
  - name: add ergadmin to docker group
    ansible.builtin.user:
      name: ergadmin
      append : yes
      groups : docker
  - name: run rancher
    ansible.builtin.shell:
      cmd: docker run -d --restart=unless-stopped -p 443:443 -p 80:80 --privileged rancher/rancher:latest
  - name: Ensure that firewalld is enabled and started
    ansible.builtin.service:
      name: firewalld
      enabled: true
      state : started
  - name: allow port 22
    ansible.builtin.shell:
      cmd: sudo firewall-cmd --permanent --add-port=22/tcp
  - name: allow port 80
    ansible.builtin.shell:
      cmd: firewall-cmd --permanent --add-port=80/tcp
  - name: allow port 443
    ansible.builtin.shell:
      cmd: firewall-cmd --permanent --add-port=443/tcp
  - name: allow port 2376
    ansible.builtin.shell:
      cmd: firewall-cmd --permanent --add-port=2376/tcp
  - name: allow port 2379
    ansible.builtin.shell:
      cmd: firewall-cmd --permanent --add-port=2379/tcp
  - name: allow port 2380
    ansible.builtin.shell:
      cmd: firewall-cmd --permanent --add-port=2380/tcp
  - name: allow port 6443
    ansible.builtin.shell:
      cmd: firewall-cmd --permanent --add-port=6443/tcp
  - name: allow port 8472
    ansible.builtin.shell:
      cmd: firewall-cmd --permanent --add-port=8472/udp
  - name: allow port 9099
    ansible.builtin.shell:
      cmd: firewall-cmd --permanent --add-port=9099/tcp
  - name: allow port 10250
    ansible.builtin.shell:
      cmd: firewall-cmd --permanent --add-port=10250/tcp
  - name: allow port 10254
    ansible.builtin.shell:
      cmd: firewall-cmd --permanent --add-port=10254/tcp
  - name: allow port 30000-32767
    ansible.builtin.shell:
      cmd: firewall-cmd --permanent --add-port=30000-32767/tcp
  - name: allow port 30000-32767/udp
    ansible.builtin.shell:
      cmd: firewall-cmd --permanent --add-port=30000-32767/udp
  - name: allow port 8080
    ansible.builtin.shell:
      cmd: firewall-cmd --permanent --add-port=8080/tcp
  - name: allow interface docker0
    ansible.builtin.shell:
      cmd: firewall-cmd --change-interface=docker0
  - name: add masquerade
    ansible.builtin.shell:
      cmd: firewall-cmd --add-masquerade --permanent
  - name: reload firewall
    ansible.builtin.shell:
      cmd: firewall-cmd --reload
  - name: restart docker
    ansible.builtin.shell:
      cmd: systemctl restart docker
# https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html
# to run this file use an the example command : ansible-playbook main.yml -f 10

```

to run the playbook change directory to correct file path and run the playbook.

> cd ~/ansible/centoss-install-rancher-management-host-playbook/  

> ansible-playbook main.yml remoteservername -kK #prompts for password if you have no ssh key installed  


## Rancher GUI - Cloud Credentials

Once the management server is installed. Generate a password for the admin user and save it. Once logged in go to the top right corner and select the user settings and select cloud credentials. These are the credentials used to connect to the vcenter, privilliges required are the administrator role in the required datacenter.

Create the cloud credentials used by Rancher to Communicate with vSphere and populate with the below example details.

> Name : Rancher vCenter Credentials   
> vCenter or ESXi Server* : vcenter.int.domain.com  
> Port: 443  
> Username : svc_rancher@vcenter.int.domain.com  
> Password :


## Rancher GUI - Node Templates

Once the cloud credentials are configured the next step is to configure the template with the configuration required for Rancher to create new kubernetes workers and masters on the VMWare platform.

Go to the node templates page and enter the details for the template.

```json
Data Center : VMWAREDATACENTERNAME
Resource Pool : host/ESXCLUSTERNAME/Resources
Datastore : DATASTORENAME
Folder : /DC/vm/Rancher/
Host: esx01.int.domain.com
Networks : vmnetwork1
```

Add the below config to the Cloud Config YAML section of the node template:
The cloud-config file is attached to a vm on boot and executed. This file creates a shell script which gathers an IP from vmware tools deamon in the network protocol profiles pool. This IP is then used to create a network adapter file which is loaded on boot. Edit the netmask and DNS fields to any values you want.
```yaml
#cloud-config
write_files:
  - path: /root/network-scripts.sh
    content: |
        #!/bin/bash
        vmtoolsd --cmd 'info-get guestinfo.ovfEnv' > /tmp/ovfenv
        IPAddress=$(sed -n 's/.*Property oe:key="guestinfo.interface.0.ip.0.address" oe:value="\([^"]*\).*/\1/p' /tmp/ovfenv)
        SubnetMask=$(sed -n 's/.*Property oe:key="guestinfo.interface.0.ip.0.netmask" oe:value="\([^"]*\).*/\1/p' /tmp/ovfenv)
        Gateway=$(sed -n 's/.*Property oe:key="guestinfo.interface.0.route.0.gateway" oe:value="\([^"]*\).*/\1/p' /tmp/ovfenv)
        cat > /etc/sysconfig/network-scripts/ifcfg-ens192 <<EOF
        BOOTPROTO=STATIC
        DEVICE=ens192
        ONBOOT=yes
        TYPE=Ethernet
        USERCTL=no
        IPADDR=$IPAddress
        NETMASK=255.255.255.0
        GATEWAY=$Gateway
        DNS1=10.2.2.100
        PREFIX=24
        DEFROUTE=Yes
        NAME=ens192
        DOMAIN=int.eurasiancorp.com
        EOF
runcmd:
- bash /root/network-scripts.sh
- ifup ens192
- echo -e 'net.bridge.bridge-nf-call-ip6tables = 1\nnet.bridge.bridge-nf-call-iptables = 1\n'>/etc/sysctl.d/10-rancher.conf 
- firewall-cmd --permanent --add-port=22/tcp
- firewall-cmd --permanent --add-port=80/tcp
- firewall-cmd --permanent --add-port=443/tcp
- firewall-cmd --permanent --add-port=1337/tcp
- firewall-cmd --permanent --add-port=2376/tcp
- firewall-cmd --permanent --add-port=2379/tcp
- firewall-cmd --permanent --add-port=2380/tcp
- firewall-cmd --permanent --add-port=4789/udp #VXLAN for flannel
- firewall-cmd --permanent --add-port=6443/tcp
- firewall-cmd --permanent --add-port=8080/tcp
- firewall-cmd --permanent --add-port=8472/udp
- firewall-cmd --permanent --add-port=9090/tcp
- firewall-cmd --permanent --add-port=10240-10260/tcp
- firewall-cmd --permanent --add-port=30000-32767/tcp
- firewall-cmd --permanent --add-port=30000-32767/udp
- firewall-cmd --permanent --add-port=40000-40500/tcp
- firewall-cmd --reload
bootcmd:
- sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
- setenforce 0
```

Enter the correct portgroup used by the VM under networks and leave other options as default.

https://www.virtualthoughts.co.uk/2020/03/29/rancher-vsphere-network-protocol-profiles-and-static-ip-addresses-for-k8s-nodes/


Enter the below vApp properties and save the node template:

```
Provide a custom vApp config
OVF environment transport: com.vmware.guestinfo
vApp IP protocol: IPv4
vApp IP allocation policy: fixedAllocated


vApp Properties:
guestinfo.interface.0.ip.0.address : ip:vmnetwork1
guestinfo.interface.0.ip.0.netmask : ${netmask:vmnetwork1}
guestinfo.interface.0.route.0.gateway : ${gateway:vmnetwork1}

Name: VMWare Rancher Template
```
