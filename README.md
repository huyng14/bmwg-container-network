# bmwg-container-network
IETF hackathon 109<br/>
*Contact: Huy Nguyen (huynq@dcn.ssu.ac.kr)*

## Physical Topology
![Physical Topology.jpg](images/physical-topology.jpg)
## Hardware specification

| Node Name | Specification  | Description |
| ------------- | ------------- | ------------- |
|Container Control<br/>Master node| - Intel(R) Core(TM) i5-6200U CPU<br/> (1socket x 4core)<br/> - MEM 8GB<br/> - DISK 500GB<br/> - Control plane: NIC 1Gb | - Container Deployment and Network Allocation<br/> - Kubernetes Master |
| Conatiner Service<br/>Worker node | - Intel(R) Xeon(R E5-2620 v3 @2.4Ghz<br/> (1socket x 6core)<br/> - MEM 128GB<br/> - DISK 2TB<br/> - Control plane: NIC 1Gb<br/> - Data plane: XL710-qda2 (1NIC 2Ports-40Gb) | - Container Service<br/> - Kubernets Worker
| Traffic Generator | - Intel(R) Xeon(R) Gold 6148 @ 2.4Ghz<br/> (2Socket X 20Core)<br/> - MEM 128G<br/> - DISK 2TB<br/> - Control plane: NIC 1Gb<br/> - Data plane: XL710-qda2 (1NIC 2Ports-40Gb) | - Traffic Generator<br/> - TREX v2.82 |

## Software components
| Software Function | Software Component | Link
| ------------- | ------------- | ------------- |
| Host OS | - Master node:<br/> &nbsp;&nbsp;&nbsp;&nbsp; *- Ubuntu 16.04.7 LTS<br/> &nbsp;&nbsp;&nbsp;&nbsp; - Kernel version 4.15.0-122<br/>* - Worker node and traffic generator:<br/> &nbsp;&nbsp;&nbsp;&nbsp; *- CentOS 7.8.2003<br/> &nbsp;&nbsp;&nbsp;&nbsp; - Kernel version 3.10.0-1127.19.1.el7.x86_64* | https://ubuntu.com/<br/> https://www.centos.org/ |
| Ansible | Ansible v2.7.16 | https://www.ansible.com/ |
| BMRA ansible playbook | Kubernetes v1.16 branch | https://github.com/intel/container-experience-kits |
| Python | Python 2.7 | https://www.python.org/ |
| Kubespray | Kubespray: v2.12 | https://github.com/kubernetes-sigs/kubespray |
| Docker* | v19.03.13 | https://www.docker.com/ |
| Kubernetes | v1.16.9 | https://github.com/kubernetes/kubernetes |
| CPU manager for Kubernetes | v1.4.0 | https://github.com/intel/CPU-Manager-for-Kubernetes |
| Data Plane Development Kit | v19.11.5 |  http://fast.dpdk.org/rel/dpdk-19.11.5.tar.xz | 
| Multus CNI | MULTUS CNI v3.3 | https://github.com/intel/multus-cni |
| SRIoV CNI | SR-IOV CNI v2.0.0 | https://github.com/intel/sriov-cni |
| SRIoV network device plugin| v3.1 | https://github.com/intel/sriov-network-device-plugin |
| Userspace CNI | v1.2 | https://github.com/intel/userspace-cni-network-plugin |
| Intel Ethernet Drivers | | https://sourceforge.net/projects/e1000/files/iavf%20stable/4.0.1<br/> https://sourceforge.net/projects/e1000/files/i40e%20stable/2.13.10 |

## Software Prerequisites for Ansible Host, Master Nodes, and Worker Nodes
1. Enter the following commands in Ansible Host:
```
# sudo su
# python -m virtualenv ansible
# source ansible/bin/activate
# pip install ansible==2.7.16
# sudo python get-pip.py
# sudo pip install ansible
# pip2 install jinja2 –upgrade
```
2. Enable login without passward between all nodes in the cluster
Step 1: Create Authentication SSH-Keygen Keys on Ansible Host:
```
# ssh-keygen
```
Step 2: Upload Generated Public Keys to all the nodes from Ansible Host:
```
# ssh-copy-id root@node-ip-address
```

## Deploy Bare Metal Architecture using Ansible Playbook
1. Get Ansible playbook:
```
# git clone https://github.com/huyng14/bmwg-container-network.git
# cd bmwg-container-network
```
2. Edit the inventory.ini to reflect the requirement. Here is the sample file:
```
[all]
master1 ansible_host=192.168.26.20 ip=192.168.26.20
node1   ansible_host=192.168.26.26 ip=192.168.26.26

[kube-master]
master1

[etcd]
master1

[kube-node]
node1

[k8s-cluster:children]
kube-master
kube-node

[calico-rr]
```
3. Update group_vars to match the desired configuration
```
---
## BMRA master playbook variables ##

# Node Feature Discovery
nfd_enabled: false
nfd_build_image_locally: false
nfd_namespace: kube-system
nfd_sleep_interval: 30s

# Intel CPU Manager for Kubernetes
cmk_enabled: true
cmk_namespace: kube-system
cmk_use_all_hosts: false # 'true' will deploy CMK on the master nodes too
cmk_hosts_list: node1 # allows to control where CMK nodes will run, leave this option commented out to deploy on all K8s nodes
cmk_shared_num_cores: 2 # number of CPU cores to be assigned to the "shared" pool on each of the nodes
cmk_exclusive_num_cores: 2 # number of CPU cores to be assigned to the "exclusive" pool on each of the nodes
cmk_shared_mode: packed # choose between: packed, spread, default: packed
cmk_exclusive_mode: packed # choose between: packed, spread, default: packed

# Intel SRIOV Network Device Plugin
sriov_net_dp_enabled: true
sriov_net_dp_namespace: kube-system
# whether to build and store image locally or use one from public external registry
sriov_net_dp_build_image_locally: false
# SR-IOV network device plugin configuration.
# For more information on supported configuration refer to: https://github.com/intel/sriov-network-device-plugin#configurations
sriovdp_config_data: |
    {
        "resourceList": [{
                "resourceName": "intel_sriov_dpdk_p1",
                "selectors": {
                    "vendors": ["8086"],
                    "devices": ["154c", "10ed"],
                    "drivers": ["vfio-pci", "iavf"],
                    "pfNames": ["ens786f1"],
                    "needVhostNet": false
                }
            },
            {
                "resourceName": "intel_sriov_dpdk_p0",
                "selectors": {
                    "vendors": ["8086"],
                    "devices": ["154c", "10ed"],
                    "drivers": ["vfio-pci", "iavf"],
                    "pfNames": ["ens786f0"],
                    "needVhostNet": false
                }
            }
        ]
    }
# Intel Device Plugins for Kubernetes
qat_dp_enabled: true
qat_dp_namespace: kube-system
gpu_dp_enabled: false
gpu_dp_namespace: kube-system

# Intel Telemetry Aware Scheduling
tas_enabled: false
tas_namespace: default
# create default TAS policy: [true, false]
tas_create_policy: false

# Create reference net-attach-def objects
example_net_attach_defs:
  userspace_ovs_dpdk: false
  userspace_vpp: false
  sriov_net_dp: false

## Proxy configuration ##
#http_proxy: "http://proxy.example.com:1080"
#https_proxy: "http://proxy.example.com:1080"
#additional_no_proxy: ".example.com"

#Topology Manager flags
kubelet_node_custom_flags:
  - "--feature-gates=TopologyManager=true"
  - "--topology-manager-policy=best-effort"

# Kubernetes cluster name, also will be used as DNS domain
cluster_name: cluster.local

## Kubespray variables ##

# default network plugins and kube-proxy configuration
kube_network_plugin_multus: true
multus_version: v3.3
kube_network_plugin: flannel
kube_pods_subnet: 10.244.0.0/16
kube_service_addresses: 10.233.0.0/18
kube_proxy_mode: iptables

# please leave it set to "true", otherwise Intel BMRA features deployed as Helm charts won't be installed
helm_enabled: true

# Docker registry running on the cluster allows us to store images not avaialble on Docker Hub, e.g. CMK
registry_enabled: true
registry_storage_class: ""
registry_local_address: "localhost:5000"
```
4. Update files in the host_vars directory to match the desired configuration
```
---
# Kubernetes node configuration

# Enable SR-IOV networking related setup
sriov_enabled: true

# sriov_nics: SR-IOV PF specific configuration list
sriov_nics:
  - name: ens786f0  # PF interface names
    sriov_numvfs: 4              # number of VFs to create for this PF(enp24s0f0)
    vf_driver: iavf          # VF driver to be attached for all VFs under this PF(enp24s0f0), "i40evf", "iavf", "vfio-pci", "igb_uio"
#    ddp_profile: "gtp.pkgo"      # DDP package name to be loaded into the NIC
  - name: ens786f1
    sriov_numvfs: 4
    vf_driver: iavf

sriov_cni_enabled: true

# install DPDK
install_dpdk: true # DPDK installation is required for sriov_enabled:true; default to false

userspace_cni_enabled: true

# Intel Bond CNI Plugin
bond_cni_enabled: false 

vpp_enabled: false
ovs_dpdk_enabled: true
# CPU mask for OVS-DPDK PMD threads
ovs_dpdk_lcore_mask: 0x1
# Huge memory pages allocated by OVS-DPDK per NUMA node in megabytes
# example 1: "256,512" will allocate 256MB from node 0 abd 512MB from node 1
# example 2: "1024" will allocate 1GB fron node 0 on a single socket board, e.g. in a VM
ovs_dpdk_socket_mem: "256,0"

# Set to 'true' to update i40e and i40evf kernel modules
force_nic_drivers_update: true

# install Intel x700 & x800 series NICs DDP packages
install_ddp_packages: false

# Enables hugepages support
hugepages_enabled: true

# Hugepage sizes available: 2M, 1G
default_hugepage_size: 1G

# Sets how many hugepages of each size should be created
hugepages_1G: 4
hugepages_2M: 12

# CPU isolation from Linux scheduler
isolcpus_enabled: true
isolcpus: "4-7"

# Intel CommsPowerManagement
sst_bf_configuration_enabled: true
# Option sst_bf_mode requires sst_bf_configuration_enabled to be set to 'true'.
# There are three configuration modes:
# [s] Set SST-BF config (set min/max to 2700/2700 and 2100/2100)
# [m] Set P1 on all cores (set min/max to 2300/2300)
# [r] Revert cores to min/Turbo (set min/max to 800/3900)
sst_bf_mode: s
```
5. Update and initialize git submodule:
```
# git submodule update --init
```
6. Execute the Ansible Playbook
```
#ansible-playbook -i inventory.ini playbooks/cluster.yml
```
