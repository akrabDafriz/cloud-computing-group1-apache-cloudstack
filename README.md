# Apache Cloudstack Private Cloud Installation & Configuration
![](https://ee.ui.ac.id/wp-content/uploads/180/2024/02/Logo-Teknik-Elektro-Mini-1-1024x461-1.png)

Contributors:
- Yoel Dwi Miryano (2206059534)
- Giovan Christoffel Sihombing (2206816084)
- Kevin Ariono (2206059603)
- Muhammad Daffa Rizkyandri (2206829194)

# Video Installation

<a href="http://www.youtube.com/watch?feature=player_embedded&v=SHd-b_ka6fQ" target="_blank">
 <img src="http://img.youtube.com/vi/SHd-b_ka6fQ/mqdefault.jpg" alt="Watch the video" width="1080" height="1920" border="10" />
</a>

# 1. Introduction

## Apache Cloudstack
Apache CloudStack is an open-source cloud computing platform designed to deploy and manage large networks of virtual machines, similar to services like Azure or AWS. Apache Cloudstack, or Cloudstack in short, adopts an Infrastructure-as-a-Service (IaaS) approach that allows users to create and manage virtualized resources through a web interface or API. Cloudstack supports multiple hypervisors, including VMware, KVM, and XenServer, offering features like multi-tenancy, high availability, and resource accounting. These features makes Cloudstack suitable for private, public, and hybrid cloud environments.

## Mechanism & Architecture
Cloudstack works by managing and provisioning virtualized resources - such as compute, storage, and networking - across one or more physical data centers to create a cloud environment:

### 1. Infrastructure Layer

Cloudstack runs on top of physical servers, which are grouped into clusters, pods, and zones. Hypervisors are utilized by these servers (KVM, XenServer, or VMware) to run virtual machines

### 2. Management Server

The management server controls the cloud environment by handling user requests, provisioning VMs, assigning storage configuring networking, and database handling of all the cloud's resources

### 3. Orchestration

Cloudstack decides where to deploy virtual machines based on available resources, policies, and templates. It communicates with the underlying hypervisor to set the VM, set up storage, and configure network access

### 4. Networking

Cloudstack sets up virtual networks, firewalls, and load balancers for VMs through a built-in network management system. It supports basic, advanced, and isolated networking setups

![](https://www.shapeblue.com/wp-content/uploads/2023/07/Apache_CloudStack_illustration.webp)
![](https://docs.cloudstack.apache.org/projects/archived-cloudstack-getting-started/en/latest/_images/region-overview.png)
![](https://infohub.delltechnologies.com/static/media/d7ece18b-2a75-4282-aa94-408c5264fdb2.png)

## Dependencies

### Database (MySQL)

MySQL is an open-source relational DBMS used to store and manage structured data. MySQL acts as the backend database that holds all configuration data, metadata, and state information about the cloud environment. This includes details about virtual machines, networks, users, storage, and system events. The CloudStack management server relies on MySQL to read and write data needed to orchestrate and manage cloud operations.

### Hypervisor (KVM)

KVM (Kernel-based Virtual Machine) is an open-source virtualization technology built into the Linux kernel that allows a physical server to run multiple isolated virtual machines (VMs). KVM acts as one of the supported hypervisors used to create and manage VMs on host machines. CloudStack communicates with KVM through tools like libvirt to provision, start, stop, and monitor VMs.

## Environment Setup

### Hardware Requirements

```
CPU: 64-bit x86 CPU
RAM: 4 GB (24 GB Recommended)
STORAGE: 250 GB (500 GB Recommended)
NETWORK: At least 1 NIC
OS: EL8 + or Ubuntu 22.04 or higher
```

### Network Address

```
Network Address: 192.168.106.0
Host IP Address: 192.168.106.155
Gateway: 192.168.106.1
Management IP: 192.168.106.155
System IP: 192.168.106.150 - 192.168.106.169
Public IP: 192.168.106.170 - 192.168.106.200
```

## Ubuntu Server Setup

### Install Basic Packages

```
apt-get install openntpd openssh-server sudo vim htop tar
```

## Network Setup

Install Bridge Utilities

```
apt-get install bridge-utils
```

Edit Network Configuration File Under /netplan Directory

```
/etc/netplan/01-netcfg.yaml
```

Netplan Configuration

```
network:
    version: 2
    renderer: networkd
    ethernets:
        eno1:
            dhcp4: false
            dhcp6: false
    bridges:
        cloudbr0:
            interfaces: [eno1]
            addresses: [192.168.106.155/23]
            routes:
                - to: default
                via: 192.168.106.1
            nameservers:
                addresses: [8.8.8.8, 1.1.1.1]
            dhcp4: false
            dhcp6: false
            parameters:
                stp: false
                forward-delay: 0
```

Apply Configuration and Reboot

```
netplan generate
netplan apply
reboot
```

## Repo Setup

Add Cloudstack & Update

```
mkdir -p /etc/apt/keyrings
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null

echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.20 / > /etc/apt/sources.list.d/cloudstack.list
apt-get update -y
```

## Management Server Setup

Install Cloudstack Management & SQL Server

```
apt-get update -y
apt-get install cloudstack-management mysql-server
```

Install Usage Server

```
apt-get install cloudstack-usage
```

MySQL Configuration

```
server_id = 1
sql-mode="STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,ERROR_FOR_DIVISION_BY_ZERO,NO_ZERO_DATE,NO_ZERO_IN_DATE,NO_ENGINE_SUBSTITUTION"
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=1000
log-bin=mysql-bin
binlog-format = 'ROW'
```

Restart MySQL Server & Setup Database

```
systemctl restart mysql
sudo systemctl status mysql
```

## Storage Setup

Install NFS Server

```
apt-get install nfs-kernel-server quota
```

Directory Export & Configuration

```
echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```

Configuration & Restart NFS Server

```
sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
echo "NEED_STATD=yes" >> /etc/default/nfs-common
sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
service nfs-kernel-server restart
```

## KVM Host Setup

Install KVM & Cloudstack Agent

```
apt-get install qemu-kvm cloudstack-agent
```

Libvirt Configuration

Change nvc_listen to "0.0.0.0"

```
vnc_listen = "0.0.0.0"
```

Activate Listen Mode in libvirtd

```
echo LIBVIRTD_ARGS=\"--listen\" >> /etc/default/libvirtd
systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```

Configure libvirt to Connect with libvirtd

```
remote_mode="legacy"
```

Configure Default libvirtd

```
listen_tls=0
listen_tcp=1
tcp_port="16509"
mdns_adv=0
auth_tcp="none"
```

Restart libvirtd

```
systemctl restart libvirtd
```

Sysctl Configuration

```
net.bridge.bridge-nf-call-arptables = 0
net.bridge.bridge-nf-call-iptables = 0

sysctl -p
net.bridge.bridge-nf-call-arptables = 0
net.bridge.bridge-nf-call-iptables = 0
```

Set Unique Host UUID

```
apt-get install uuid
```

## Firewall

Use UFW to Configure Firewall

```
ufw allow mysql
```

Non-Activate AppArmor for libvirtd

```
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

## Management Server

Start your Cloud

```
cloudstack-setup-management
systemctl status cloudstack-management
tail -f /var/log/cloudstack/management/management-server.log
```

## Cloudstack Setup

### Zone Configuration

Cloudstack Dashboard

![](https://i.imgur.com/fnqxouk.png)

Secondary Dashboard Panel

![](https://i.imgur.com/pcpx4p2.png)

Configuration Tab

![](https://i.imgur.com/z2MpvMy.png)

Zone Configuration

![](https://i.imgur.com/dGSZH3x.png)

Cloud Network Configuration

![](https://i.imgur.com/5UhJwtz.png)

IP Setting

![](https://i.imgur.com/OuhRkYv.png)

System Resources Summary

![](https://i.imgur.com/MBPnT3L.png)

Volume Tab - VM View

![](https://i.imgur.com/v5Eo8f7.png)

VM Networking Detail

![](https://i.imgur.com/DPiY8mB.png)

VM INstance Overview

![](https://i.imgur.com/v0D3h2o.png)

Virtual Router Details

![](https://i.imgur.com/U9e2vgN.png)

Public IP & NAT Config

![](https://i.imgur.com/lBbFv3p.png)

Network Rules

![](https://i.imgur.com/UN40ZHN.png)

Firewall Rules Overview

![](https://i.imgur.com/SsFijIP.png)


# 5. Complete Video Tutorial
https://youtu.be/SHd-b_ka6fQ?si=tZMGL5VbddFDQ1SY

# 6. References

1. https://docs.cloudstack.apache.org/en/latest/installguide/building_from_source.html
2. https://docs.cloudstack.apache.org/en/latest/installguide/building_from_source.html
3. https://qa.cloudstack.cloud/builds/docs-build/pr/348/installguide/management-server/index.html
