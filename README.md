# Proxmox Linstor Talos HCI Cluster
## Introduction
A fully working 02 nodes HCI (Hyper Converged Infrastructure) with 3rd node (as witness + backup server) which can host both VM (Virtual Machine) and Kubernetes workload based on:
- Promox (Compute)
- Linstor (Storage)
- Talos Linux (Kubernetes Cluster)
- Cilium (CNI plugin for Kubernetes)
## Proxmox Installation
* Hardware Nodes
  The test environment consists of 03 HPE servers
  - Proxmox is installed on a boot device `/dev/sda`
  - The rest of the disks used for Linstor in RAID 10 setup
  - Network overview  
    ![image](https://github.com/user-attachments/assets/eb9bcdd1-eb65-4dc2-b7d5-c277c0a5b9f4)

* Setup Proxmox Network
  To make the setup more light-weight, management and VM traffic will be combined 
  - **Management Network & VM Traffic**: 2x 1Gbps connections in LACP mode
  - **DRBD Sync Traffic**: 2 x 10Gbps connections. To improve synchronization performance, RDMA will be used. 
  ![image](https://github.com/user-attachments/assets/d385b586-51b2-48c9-96cc-3db03c5a786c)

* Add 03 nodes to cluster
  Configure Proxmox cluster is a very straight forward process. Link to officical document: https://pve.proxmox.com/wiki/Cluster_Manager
  ![image](https://github.com/user-attachments/assets/dc9a67e2-211a-4913-abf5-9b76d79fc868)
**Note:** The cluster consists of 03 nodes but only 02 nodes will have DRBD Storage Sync, third node is only for witness  

* Configure HA  
  ![image](https://github.com/user-attachments/assets/a1e33a38-a07d-4c83-bbe8-d0283ec2d891)

## Linstor Installation
Linstor installation is available here: https://pve.proxmox.com/wiki/Deploy_Hyper-Converged_Ceph_Cluster](https://linbit.com/blog/setting-up-highly-available-storage-for-proxmox-using-linstor-the-linbit-gui/?utm_term=&utm_campaign=PM+%7C+US&utm_source=adwords&utm_medium=ppc&hsa_acc=4220519389&hsa_cam=20773050617&hsa_grp=&hsa_ad=&hsa_src=x&hsa_tgt=&hsa_kw=&hsa_mt=&hsa_net=adwords&hsa_ver=3&gad_source=1&gad_campaignid=21195166738&gclid=Cj0KCQjwmK_CBhCEARIsAMKwcD7N0_7A1CZ8ROq0k6rHyE67DUJqUlxXTalTFIA_rcEWAkVufrxqu1waArLnEALw_wcB)  
* Storage Pools:  
  `proxmox-storage`: for storing VM disks  
  `k8s-storage`: providing persistent volume for Kubernetes cluster  
* Linstor overview  
 ![image](https://github.com/user-attachments/assets/82517d24-2852-4766-8515-a953b83a38b1)

* RDMA Setup
````
  # Install necessary RDMA packages
  apt -y install rdma-core
  apt -y install infiniband-diags ibverbs-providers ibverbs-utils
  # Following commands can be used to check supported RDMA devices
  ibv_devices
    device                 node GUID
    ------              ----------------
    rocep18s0f0         5eba2cfffeb52f9c
    rocep18s0f1         5eba2cfffeb52f9d
    rocep55s0f0         5eba2cfffe62b4f0
    rocep55s0f1         5eba2cfffe62b4f8
  # Test RDMA traffic before using it for DRBD Sync
  # On one of the node - server side
  ib_send_bw -d {rdma_interface} -i 1 -F --report_gbits
  
  ************************************
  * Waiting for client to connect... *
  ************************************
  # On one of the node - client side
  ib_send_bw -d {rdma_interface} -i 1 -F --report_gbits {server_ip}
  
---------------------------------------------------------------------------------------
                    Send BW Test
 Dual-port       : OFF          Device         : rocep18s0f0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: ON
 ibv_wr* API     : OFF
 TX depth        : 128
 CQ Moderation   : 1
 Mtu             : 4096[B]
 Link type       : Ethernet
 GID index       : 3
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0xff0000 PSN 0xe4233a
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:01:03
 remote address: LID 0000 QPN 0xff02fe PSN 0xdbbe61
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:01:02
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
 65536      1000             9.81               9.81               0.018715
---------------------------------------------------------------------------------------
````
### Change DRBD to use RDMA for Synchronization on cluster node 01 and 02
````
# Add RDMA interfaces to cluster nodes
linstor node interface create {node_name} rdma1 {rdma_nic_ip}
linstor node interface create {node_name} rdma1 {rdma_nic_ip}
linstor node interface create {node_name} rdma2 {rdma_nic_ip}
linstor node interface create {node_name} rdma2 {rdma_nic_ip}
# Use the following script to update the resources with two RDMA sync paths
#!/bin/bash

# Read the output of linstor command
resources=$(linstor --no-utf8 resource list  | awk 'NR>3 {print $2}' | sort -u)

for res in $resources; do
    echo "Processing resource: $res"
    linstor resource-connection path create {node1_name} {node2_name} $res path1 rdma1 rdma1
    linstor resource-connection path create {node1_name} {node2_name} $res path2 rdma2 rdma2
done
````

## Kubernetes Cluster Installation ##
Following VMs are used for K8s cluster:
````
# Control Planes
talos-cp-01
talos-cp-02
talos-cp-03
# Worker Nodes
talos-wk-01
talos-wk-02
talos-wk-03
````
Talos Linux is used due to its declarative (config via .yaml file), security (no direct shell access)  
### Preparation ###
* Talos ISO  
Since Talos will be running as a VM on Proxmox, qemu agent and drbd plugin need to be added to be able to see the information from Proxmox and connect to Linstor for persistent storage. This is optional. Talos ISO can be generated from https://factory.talos.dev  
![image](https://github.com/user-attachments/assets/a2c4e250-a948-4d1f-835a-6e145e62c581)  
Record the image ID and download the iso  
Bootup the servers using talos iso.  
![image](https://github.com/user-attachments/assets/4ecf3e36-a3cc-4461-ada7-ea61b70ed00d)  

* Client Tools
  - `talosctl` https://www.talos.dev/v1.9/talos-guides/install/talosctl/
  - `kubectl` https://kubernetes.io/docs/tasks/tools/
  - `helm` https://helm.sh/docs/intro/install/
  - `k9s`(optional) https://k9scli.io/topics/install/
### Bootstrap K8s Cluster ###
* Separating out secrets  
When generating the configuration files for a Talos Linux cluster, it is recommended to start with generating a secrets bundle which should be saved in a secure location. This bundle can be used to generate machine or client configurations at any time:  
`talosctl gen secrets -o secrets.yaml`  

* Generate the machine configuration for each node:  
  `talosctl gen config --with-secrets secrets.yaml <cluster-name> <k8s-cluster-endpoint> `  
  `For example: talosctl gen config --with-secrets .\secrets.yaml kubevirt-talos https://172.49.172.199:6443`

  The command above will generate `controlplane.yaml` `worker.yaml` `talsconfig`  
  Copy and edit control plane files (IPs, ISO ID, POD subnets,...) according to your situation and design
  - `talos-cp-01.yaml`
  - `talos-cp-02.yaml`
  - `talos-cp-03.yaml`
  - `talos-wk-01.yaml`
  - `talos-wk-01.yaml`
  - `talos-wk-02.yaml`
  - `talos-wk-03.yaml`

* Apply config on first node
  ```
  talosctl apply -f talos-cp-01.yaml -n {ip_of_node_01} --insecure
  # Node will be rebooted
  # Bootstrap k8s cluster
  talosctl bootstrap -n {ip_of_node_01}
  ```
* When node-01 is in healthy status, the config can be applied to remaining nodes
  ````
  # Node 02
  talosctl apply -f talos-cp-02.yaml -n {ip_of_node_02}
  # Node 03
  talosctl apply -f talos-cp-03.yaml -n {ip_of_node_03}
  ````
  ![image](https://github.com/user-attachments/assets/f3e6d66a-6d36-4fe1-a25b-8f4b83395006)  
  Don't worry about Ready in False state at this stage, CNI will be deployed at a later step  
* Generate kubeconfig file  
  For ease of management when there are multiple clusters to manage, we will use contexts
  ````
  # Add talos context
  talosctl config add {context_name}
  # Set the current context
  talosctl config context {context_name}
  # Generate kubeconfig file
  talosctl kubeconfig --talosconfig=./talosconfig -n {vip_of_the_cluster}
  ````
  We can now list the nodes with `kubectl get nodes`  
* Install Cilium CNI
  ````
  # Add cilium helm repo
  helm repo add cilium https://helm.cilium.io/
  helm repo update
  helm install cilium cilium/cilium `
  --namespace kube-system `
  --set ipam.mode=kubernetes `
  --set ipv4NativeRoutingCIDR=172.248.0.0/16 ` # Change according to your design
  --set hubble.enabled=true `
  --set hubble.relay.enabled=true `
  --set hubble.ui.enabled=true `
  --set kubeProxyReplacement=true `
  --set enable-bpf-masquerade=true `
  --set enable-ipv4-masquerade=true `
  --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" `
  --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" `
  --set cgroup.autoMount.enabled=false `
  --set cgroup.hostRoot=/sys/fs/cgroup `
  --set k8sServiceHost=localhost `
  --set k8sServicePort=7445
  ```` 
* iBGP routing
  To be able to access pods from external and reduce complexity of using a load balancer, iBGP will be used with CNI to advertise service IPs.
  ````
  # Enable BGP and direct routing mode since our nodes are in the same L2 domain
  helm upgrade cilium cilium/cilium --namespace kube-system --reuse-values --set bgpControlPlane.enabled=true --set routingMode=native --set autoDirectNodeRoutes=true --set loadBalancer.mode=dsr
  kubectl -n kube-system rollout restart ds/cilium
  kubectl apply -f .\cilium-bgp-peer.yaml
  kubectl apply -f .\cilium-bgp.yaml
  kubectl apply -f .\cilium-bgp-adv.yaml
  # Expose Cilium Hubble UI service
  kubectl delete service/hubble-ui -n kube-system
  kubectl expose deployment hubble-ui --type="LoadBalancer" --port 80 --target-port=8081 -n kube-system
  ````
  Hubble UI should be accessible from network
  ![image](https://github.com/user-attachments/assets/7217c4c0-947a-404f-9f5b-c0100ef83ed5)
  ![image](https://github.com/user-attachments/assets/4914a00a-a2ca-44d9-af14-470f5da4ef41)

### Deploy Linstor CSI  ###
````
  # Official guide: https://piraeus.io/docs/stable/tutorial/get-started/#prerequisites
  # Install piraeus operator
  kubectl apply --server-side -k "https://github.com/piraeusdatastore/piraeus-operator//config/default?ref=v2.8.1" 
  # Apply DRBD-parameters for Talos Linux
  kubectl apply -f .\talos-loader-override.yaml
  # Install kubectl-linstor plugin
  https://github.com/piraeusdatastore/kubectl-linstor
  # Deploy linstor pods
  kubectl apply -f .\linstor-cluster.yaml
  ````
  After 10-15 minutes, the pods should all be in running status and ready for next steps
 ![image](https://github.com/user-attachments/assets/db4cb0ce-790a-4953-988b-e86084d5717c)

````
# Use host network to connect to Linstor controller
kubectl apply -f .\linstor-host-net.yaml
# Create storage class
kubectl apply -f .\linstor-sc.yaml
# Create persistent volume claim
kubectl apply -f .\linstor-pv.yaml
````
Storage class is created and test PVC in bound status
![image](https://github.com/user-attachments/assets/06725a8c-203b-4b2a-aa08-9f98a63297bb)
![image](https://github.com/user-attachments/assets/79079704-e081-45f5-99dd-9c650e96d8b4)










