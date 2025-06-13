# Proxmox Talos HCI Cluster
## Introduction
A fully working 03 nodes HCI (Hyper Converged Infrastructure) which can host both VM (Virtual Machine) and Kubernetes workload based on:
- Promox (Compute)
- Ceph (Storage - Included with Proxmox)
- Talos Linux (Kubernetes Cluster)
- Cilium (CNI plugin for Kubernetes)
## Proxmox Installation
* Hardware Nodes
  The test environment consists of 03 HPE servers
  - Proxmox is installed on a boot device `/dev/sda`
  - The rest of the disks used for Ceph is configured in HBA mode (pass-through mode)
  - Network overview  
    ![image](https://github.com/user-attachments/assets/99f8a3ef-9c04-4f60-b352-bdeb30cf6e6b)  
* Setup Proxmox Network  
  - **Management Network**: 2x 1Gbps connections in active-passive mode
  - **Ceph Public**: 2 x 10Gbps connections in LACP mode. This network is where the clients (k8s cluster in this case) will connect for persistence volume.  
  - **VM Traffic***: 2 x 10Gbps connections in LACP mode.  
  ![image](https://github.com/user-attachments/assets/95a3b3a0-f92d-4272-84b3-69905d780944)  
* Add 03 nodes to cluster
  Configure Proxmox cluster is a very straight forward process. Link to officical document: https://pve.proxmox.com/wiki/Cluster_Manager
  ![image](https://github.com/user-attachments/assets/83dadb0a-eb40-4932-863a-24cc07849179)
* Configure HA  
  ![image](https://github.com/user-attachments/assets/74f7c090-a64f-4397-8fcf-e042229a963b)
## Ceph Installation  
Configure Ceph cluster is pretty easy to follow: https://pve.proxmox.com/wiki/Deploy_Hyper-Converged_Ceph_Cluster  
* Storage Pools:  
  `vm-storage`: for storing VM disks  
  `k8s-storage`: providing persistent volume for Kubernetes cluster  
* OSDs:
  ![image](https://github.com/user-attachments/assets/018eb10f-2e84-46bb-a4f2-c68d5133e182)

* Monitor:
  ![image](https://github.com/user-attachments/assets/532f8841-1ff0-4016-b50d-b5495398eb59)  

At this stage, all nodes should see ceph storage pools  
![image](https://github.com/user-attachments/assets/7d615686-6b11-4a44-ac92-f2de24b3a0de)

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
Since Talos will be running as a VM on Proxmox, qemu agent needs to be added to be able to see the information from Proxmox. This is optional. Talos ISO can be generated from https://factory.talos.dev  
![image](https://github.com/user-attachments/assets/dbd968e0-4d5f-411e-972f-0c426202a141)  
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
  kubectl apply -f .\cilium-bgp-peer.yaml
  kubectl apply -f .\cilium-bgp-pod.yaml
  kubectl apply -f .\cilium-bgp-adv.yaml
  # Expose Cilium Hubble UI service
  kubectl delete service/hubble-ui -n kube-system
  kubectl expose deployment hubble-ui --type="LoadBalancer" --port 80 --target-port=8081 -n kube-system
  ````
  Hubble UI should be accessible from network
  ![image](https://github.com/user-attachments/assets/015bc0ea-3552-42ee-b48c-30483aafca5b)  
  ![image](https://github.com/user-attachments/assets/da4c372b-ab50-4d4c-811b-ff011f095410)
### Deploy Ceph CSI RBD for K8s ###
````
# Create an authentication key
ceph auth get-or-create client.{key_name} mon 'profile rbd' osd 'profile rbd pool={ceph_pool}' mgr 'profile rbd pool={ceph_pool}'
# Record the key
# Collect Ceph monitors info
ceph mon dump
# It will show IPs of ceph monitors
epoch 3
fsid 567056cb-ecaa-4176-96d1-6dc007679b7e
last_changed 2025-05-10T21:14:07.043548+0200
created 2025-05-10T20:51:04.180710+0200
min_mon_release 19 (squid)
election_strategy: 1
0: [v2:172.49.172.184:3300/0,v1:172.49.172.184:6789/0] mon.srv01
1: [v2:172.49.172.185:3300/0,v1:172.49.172.185:6789/0] mon.srv02
2: [v2:172.49.172.186:3300/0,v1:172.49.172.186:6789/0] mon.srv03
dumped monmap epoch 3
````
Update rollout files according to your design. In this setup, Ceph CSI plugin will be installed under namespace ceph-csi-rbd
````
# After updating the file, apply the files in order
# Official guide: https://docs.ceph.com/en/latest/rbd/rbd-kubernetes/
1. csi-provisioner-rbac.yaml
2. csi-nodeplugin-rbac.yaml
3. ceph-csi-config.yaml
4. ceph-config.yaml
5. ceph-csi-encryption-kms-config.yaml
6. ceph-csi-rbd-secret
7. csi-rbdplugin-provisioner.yaml
8. csi-rbdplugin.yaml
# Create StorageClass
kubectl apply -f ceph-rbd-sc.yaml
kubectl apply -f ceph-rbd-pvc
````
Storage class is created and test PVC in bound status
![image](https://github.com/user-attachments/assets/f87ba8c4-ab50-4ddc-8610-4cf40e7504e2)  
![image](https://github.com/user-attachments/assets/832c435e-b7f6-48ab-a316-23041c083186)  

## Bonus ##
Google has recently released `kubectl-ai` to help managing and learning K8s much more interactive and easier.
````
# Install kubectl-ai
# You can easily install kubectl-ai from https://github.com/GoogleCloudPlatform/kubectl-ai
# If you have paid version OpenAI, Grok,...you can create api key and start using the tool.
# Google also provide a trial https://aistudio.google.com/
````
After the installation, you can run the tool with `kubectl-ai --model gemini-2.5-flash-preview-04-17` and start interacting with it.  
A health check question:  
![image](https://github.com/user-attachments/assets/67a29aef-245a-4502-817f-7244726f508a)  
A more advanced question:
![image](https://github.com/user-attachments/assets/cfd01b46-c263-4ce0-aa91-7f017c8e71ae)  
On the first try, the model try a command but can not find a result.
![image](https://github.com/user-attachments/assets/0bd04c3c-de32-4410-9c6d-0d182b19c049)  
It starts "thinking" and try further:
![image](https://github.com/user-attachments/assets/79b4a92c-d2da-4a9e-b14e-ea51c2572577)  
The tool can also generate and apply configuration:  
![image](https://github.com/user-attachments/assets/c4bc25eb-4715-47db-b083-208051f11104)  
![image](https://github.com/user-attachments/assets/cb983c28-e998-4425-94d7-176e4003946b)  
![image](https://github.com/user-attachments/assets/214c9395-17d2-4074-8969-bcd828c08ced)  
![image](https://github.com/user-attachments/assets/be2b3b29-ab00-41c9-8b54-5072b221a63a)








