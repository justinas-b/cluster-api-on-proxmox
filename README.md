# cluster-api-on-proxmox

## Creating VM template

Before you can proceed with deploying kubernetes cluster, you need to pre-create VM template which will be used for spinning up new kubernetes nodes. There is a tool which can help you with building those images - [image-builder](https://image-builder.sigs.k8s.io/capi/providers/proxmox), however if you would like to build it yourself, below is the guide. Please keep in mind that you need to have a separate image for each kubernetes version!

Below snippet should run on proxmox node directly:

```sh
#!/bin/bash

# https://raw.githubusercontent.com/nallej/MyJourney/main/scripts/myTemplateBuilder.sh?ref=homelab.casaursus.net
# https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
# https://kubernetes.io/docs/setup/production-environment/container-runtimes/

imageName="noble-server-cloudimg-amd64.img"
imageURL="https://cloud-images.ubuntu.com/noble/current/$imageName"
kubernetesVersion="v1.30"
volumeName="vmdata"
virtualMachineId="8000"
templateName="ubuntu-cloudimage"
tmp_cores="2"
tmp_memory="2048"
rootPasswd='Pa$$w0rd'

apt update
apt install libguestfs-tools -y
rm *.img
wget -O $imageName $imageURL
qm destroy $virtualMachineId
virt-customize -a $imageName --install qemu-guest-agent,apt-transport-https,ca-certificates,curl,gpg,socat,conntrack,net-tools,vim,jq
virt-customize -a $imageName --firstboot-command 'systemctl enable qemu-guest-agent'
virt-customize -a $imageName --root-password password:$rootPasswd

virt-customize -a $imageName --firstboot-command 'sudo apt-get update && sudo apt-get upgrade'
virt-customize -a $imageName --firstboot-command 'sudo swapoff -a'

virt-customize -a $imageName --firstboot-command 'echo -e "overlay\nbr_netfilter" > /etc/modules-load.d/k8s.conf'
virt-customize -a $imageName --firstboot-command 'sed -i -e "/#net.ipv4.ip_forward=1/c\net.ipv4.ip_forward=1" /etc/sysctl.conf'
virt-customize -a $imageName --firstboot-command 'echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.d/k8s.conf'
virt-customize -a $imageName --firstboot-command 'echo "net.bridge.bridge-nf-call-iptables  = 1" >> /etc/sysctl.d/k8s.conf'
virt-customize -a $imageName --firstboot-command 'echo "net.ipv4.ip_forward                 = 1" >> /etc/sysctl.d/k8s.conf'
virt-customize -a $imageName --firstboot-command 'echo 1 > /proc/sys/net/ipv4/ip_forward'

virt-customize -a $imageName --install containerd,runc
virt-customize -a $imageName --mkdir /etc/containerd
virt-customize -a $imageName --firstboot-command 'containerd config default | sudo tee /etc/containerd/config.toml'
virt-customize -a $imageName --firstboot-command 'sed -i "s/^\( *SystemdCgroup = \)false/\1true/" /etc/containerd/config.toml'
virt-customize -a $imageName --firstboot-command 'modprobe overlay'
virt-customize -a $imageName --firstboot-command 'modprobe br_netfilter'
virt-customize -a $imageName --firstboot-command 'systemctl --system'
virt-customize -a $imageName --firstboot-command 'systemctl restart containerd'


virt-customize -a $imageName --firstboot-command 'curl -fsSL https://pkgs.k8s.io/core:/stable:/$kubernetesVersion/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg'
virt-customize -a $imageName --firstboot-command 'echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$kubernetesVersion/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list'
virt-customize -a $imageName --firstboot-command 'sudo apt-get update && sudo apt install -y kubeadm kubectl kubelet'
virt-customize -a $imageName --firstboot-command 'apt-mark hold kubelet kubeadm kubectl'
virt-customize -a $imageName --firstboot-command 'sudo systemctl enable --now kubelet'

qm create $virtualMachineId --name $templateName --memory $tmp_memory --cores $tmp_cores --net0 virtio,bridge=vmbr1100,firewall=1
qm importdisk $virtualMachineId $imageName $volumeName
qm set $virtualMachineId --scsihw virtio-scsi-pci --scsi0 $volumeName:vm-$virtualMachineId-disk-0
qm set $virtualMachineId --boot "order=scsi0;net0" --bootdisk scsi0
qm set $virtualMachineId --serial0 socket --vga serial0
qm set $virtualMachineId --ipconfig0 ip=dhcp
qm set $virtualMachineId --agent 1
qm set $virtualMachineId --ostype l26 
qm set $virtualMachineId --tags $imageName,$kubernetesVersion,cluster-api,kubernetes
qm set $virtualMachineId --description "Image base: $imageName image<br>Kubernetes version: $kubernetesVersion<br>Used by: cluster-api"
qm template $virtualMachineId
```

If you would like to build image using [image-builder](https://image-builder.sigs.k8s.io/capi/providers/proxmox), you can run below snippet

```sh
#!/bin/bash

git clone git@github.com:kubernetes-sigs/image-builder.git
cd image-builder/images/capi

export PROXMOX_URL='https://pve.mydomain.net/api2/json'
export PROXMOX_USERNAME='svc-pve-01@MYDOMAIN.NET!aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee'
export PROXMOX_TOKEN='11111111-2222-3333-4444-555555555555'
export PROXMOX_NODE="pve"
export PROXMOX_ISO_POOL="local"
export PROXMOX_BRIDGE="vmbr1100"
export PROXMOX_STORAGE_POOL="vmdata"
export PATH=$PWD/.bin:$PATH
make deps-proxmox
make build-proxmox-ubuntu-2204
```

For `image-builder` to work, please ensure:
- that there is a DHCP server availabel in proxmox network, so that temporary VM would be automatically assigned an IP address
- that temporary VM can access server where `packer` build is initiated (ideally run packer on proxmox in same network)

## Bootstrap management cluster

### Deploy temporary `kind` cluster

In order to bootstrap management cluster, you need a temporary cluster which can be destroyed afterwards. For this we will use `kind`:

```sh
kind create cluster --name capi-bootstrap
```

### Set `proxmox` related configuration variables

Once you have temporary cluster, you need to set configuration:

```sh
## -- Controller settings -- ##
export PROXMOX_URL="https://pve.mydomain.net"                                        # The Proxmox VE host
export PROXMOX_TOKEN='svc-pve-01@MYDOMAIN.NET!aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee'  # The Proxmox VE TokenID for authentication
export PROXMOX_SECRET='11111111-2222-3333-4444-555555555555'                         # The secret associated with the TokenID

## -- Required workload cluster default settings -- ##
export PROXMOX_SOURCENODE="pve-01"                  # The node that hosts the VM template to be used to provision VMs
export TEMPLATE_VMID="8000"                         # The template VM ID used for cloning VMs
export ALLOWED_NODES="[pve-01]"                     # The Proxmox VE nodes used for VM deployments
export VM_SSH_KEYS="ssh-ed25519 ...."               # The ssh authorized keys used to ssh to the machines.

## -- networking configuration-- ##
export CONTROL_PLANE_ENDPOINT_IP="10.11.1.100"      # The IP that kube-vip is going to use as a control plane endpoint
export NODE_IP_RANGES="[10.11.1.101-10.11.1.150]"   # The IP ranges for Cluster nodes
export GATEWAY="10.11.1.1"                          # The gateway for the machines network-config.
export IP_PREFIX="24"                               # Subnet Mask in CIDR notation for your node IP ranges
export DNS_SERVERS="[10.1.1.10,10.1.1.11]"          # The dns nameservers for the machines network-config.
export BRIDGE="vmbr1100"                            # The network bridge device for Proxmox VE VMs
export POD_CIDR='172.16.0.0/16'
export SVC_CIDR='192.168.128.0/17'

## -- nodes size -- ##
export BOOT_VOLUME_DEVICE="scsi0"                   # The device used for the boot disk.
export BOOT_VOLUME_SIZE="50"                        # The size of the boot disk in GB.
export NUM_SOCKETS="2"                              # The number of sockets for the VMs.
export NUM_CORES="1"                                # The number of cores for the VMs.
export MEMORY_MIB="4096"                            # The memory size for the VMs.

export EXP_CLUSTER_RESOURCE_SET="true"              # This enables the ClusterResourceSet feature that we are using to deploy CNI
export CLUSTER_TOPOLOGY="true"                      # This enables experimental ClusterClass templating
export CLUSTER_NAME='management'
```

### Deploy controller which will be used for bootstraping

Now we can deploy [cluster-api](https://github.com/kubernetes-sigs/cluster-api) and [capmox](https://github.com/ionos-cloud/cluster-api-provider-proxmox) provider with Helm plugin:

```sh
clusterctl init --infrastructure proxmox:v0.5.1 --ipam in-cluster --core cluster-api:v1.7.2 --addon helm
```

### Deploy helm charts

Deploy below CRDs to your temporary cluster that will make sure that you have Cilium CNI and metrics-server pre-installed on your new cluster which will be created on proxmox:

```sh
kubectl apply -f - <<EOF
---
apiVersion: addons.cluster.x-k8s.io/v1alpha1
kind: HelmChartProxy
metadata:
  name: cilium
spec:
  clusterSelector:
    matchLabels: {}
  releaseName: cilium
  repoURL: https://helm.cilium.io/
  chartName: cilium
  namespace: kube-system
  valuesTemplate: |
    cluster:
      id: 0
      name: default
      k8sClientRateLimit:
        qps: "5"
        burst: "10"
---
apiVersion: addons.cluster.x-k8s.io/v1alpha1
kind: HelmChartProxy
metadata:
  name: metrics-server
spec:
  clusterSelector:
    matchLabels: {}
  releaseName: metrics-server
  repoURL: https://kubernetes-sigs.github.io/metrics-server/
  chartName: metrics-server
  namespace: kube-system
---
EOF
```

### Deploy new cluster on proxmox

Below command will generate necessary resources that will instruct cluster-api to provision new cluster:

```

clusterctl generate cluster $CLUSTER_NAME -n $CLUSTER_NAME \
  --infrastructure proxmox \
  --kubernetes-version v1.29.6 \
  --control-plane-machine-count 1 \
  --worker-machine-count 1 > ./${CLUSTER_NAME}-cluster.yaml
yq -i 'select(.kind == "Cluster").spec.clusterNetwork.pods.cidrBlocks[] = strenv(POD_CIDR)' ./${CLUSTER_NAME}-cluster.yaml
yq -i 'select(.kind == "Cluster").spec.clusterNetwork += {"services": {"cidrBlocks": [strenv(SVC_CIDR)]}}' ./${CLUSTER_NAME}-cluster.yaml
kubectl apply -n ${CLUSTER_NAME} -f ./${CLUSTER_NAME}-cluster.yaml
```

### Connecting to cluster

You can track provisioning progress using below command:
```
❯ clusterctl describe cluster ${CLUSTER_NAME}
NAME                                                          READY  SEVERITY  REASON  SINCE  MESSAGE
Cluster/capi-mgmt                                             True                     10s
├─ClusterInfrastructure - ProxmoxCluster/capi-mgmt            True                     16m
├─ControlPlane - KubeadmControlPlane/capi-mgmt-control-plane  True                     10s
│ └─Machine/capi-mgmt-control-plane-jmrfs                     True                     13s
└─Workers
  └─MachineDeployment/capi-mgmt-workers                       True                     8m30s
    └─2 Machines...                                           True                     10m    See capi-mgmt-workers-mq427-ch6bq, capi-mgmt-workers-mq427-cvtkq
```

Once your cluster is provisioned, you can retrieve `kubectl` as shown below and interact with your newly created kubernetes cluster which runs on proxmox:

```sh
❯ clusterctl get kubeconfig capi-mgmt > kubeconfig
❯ KUBECONFIG=./kubeconfig kubectl get po -A
NAMESPACE     NAME                                                    READY   STATUS    RESTARTS        AGE
kube-system   cilium-dg8v8                                            1/1     Running   1 (2m29s ago)   14m
kube-system   cilium-dqqhn                                            1/1     Running   0               12m
kube-system   cilium-operator-65496b9554-mppqf                        1/1     Running   0               14m
kube-system   cilium-operator-65496b9554-r8h6g                        1/1     Running   1 (2m29s ago)   14m
kube-system   cilium-qftwh                                            1/1     Running   0               11m
kube-system   coredns-7db6d8ff4d-8v65j                                1/1     Running   1 (2m29s ago)   14m
kube-system   coredns-7db6d8ff4d-th9wk                                1/1     Running   1 (2m29s ago)   14m
kube-system   etcd-capi-mgmt-control-plane-jmrfs                      1/1     Running   1 (2m29s ago)   14m
kube-system   kube-apiserver-capi-mgmt-control-plane-jmrfs            1/1     Running   1 (2m29s ago)   14m
kube-system   kube-controller-manager-capi-mgmt-control-plane-jmrfs   1/1     Running   1 (2m29s ago)   14m
kube-system   kube-proxy-45m4w                                        1/1     Running   0               12m
kube-system   kube-proxy-765mv                                        1/1     Running   0               11m
kube-system   kube-proxy-q7xpt                                        1/1     Running   1 (2m29s ago)   14m
kube-system   kube-scheduler-capi-mgmt-control-plane-jmrfs            1/1     Running   1 (2m29s ago)   14m
kube-system   kube-vip-capi-mgmt-control-plane-jmrfs                  1/1     Running   1 (2m29s ago)   14m
kube-system   metrics-server-7998667b79-2mr4k                         0/1     Running   0               2m30s
```

## Deploying ephemeeral clusters

Set necessary environment variables
```sh
## -- Controller settings -- ##
export PROXMOX_URL="https://pve.mydomain.net"                                        # The Proxmox VE host
export PROXMOX_TOKEN='svc-pve-01@MYDOMAIN.NET!aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee'  # The Proxmox VE TokenID for authentication
export PROXMOX_SECRET='11111111-2222-3333-4444-555555555555'                         # The secret associated with the TokenID


## -- Required workload cluster default settings -- ##
export PROXMOX_SOURCENODE="pve-01"                        # The node that hosts the VM template to be used to provision VMs
export TEMPLATE_VMID="118"                                # The template VM ID used for cloning VMs
export ALLOWED_NODES="[pve-01]"                           # The Proxmox VE nodes used for VM deployments
export VM_SSH_KEYS="ssh-ed25519 ...."                     # The ssh authorized keys used to ssh to the machines.

## -- networking configuration-- ##
export CONTROL_PLANE_ENDPOINT_IP="10.11.1.100"            # The IP that kube-vip is going to use as a control plane endpoint
export NODE_IP_RANGES="[10.11.1.101-10.11.1.105]"         # The IP ranges for Cluster nodes
export LOAD_BALANCER_RANGES="10.11.1.106-10.11.1.109"     # IP Range for load-balance IP addresses
export PDNS_ZONE="http://my.dns.zone"                     # Custom DNS zone where ephemeral clusters will provision DNS records (API endpoint)
export PDNS_API_KEY="*************"                       # API Key for authentication with DNS zone
export GATEWAY="10.11.1.1"                                # The gateway for the machines network-config.
export IP_PREFIX="24"                                     # Subnet Mask in CIDR notation for your node IP ranges
export DNS_SERVERS="[10.1.1.10,10.1.1.11]"                # The dns nameservers for the machines network-config.
export BRIDGE="vmbr1100"                                  # The network bridge device for Proxmox VE VMs

## -- xl nodes -- ##
export BOOT_VOLUME_DEVICE="scsi0"                         # The device used for the boot disk.
export BOOT_VOLUME_SIZE="50"                              # The size of the boot disk in GB.
export NUM_SOCKETS="2"                                    # The number of sockets for the VMs.
export NUM_CORES="1"                                      # The number of cores for the VMs.
export MEMORY_MIB="4096"                                  # The memory size for the VMs.

export EXP_CLUSTER_RESOURCE_SET="true"                    # This enables the ClusterResourceSet feature that we are using to deploy CNI
export CLUSTER_TOPOLOGY="true"                            # This enables experimental ClusterClass templating

export POD_CIDR='172.16.0.0/16'
export SVC_CIDR='192.168.128.0/17'
export CLUSTER_NAME='ephemeral1'
```

Create dedicated kubernetes namespace for ephemeral cluster and deploy helm chart proxies to that namespace

```sh
kubectl create ns $CLUSTER_NAME
kubectl apply -n $CLUSTER_NAME -f - <<EOF
---
apiVersion: addons.cluster.x-k8s.io/v1alpha1
kind: HelmChartProxy
metadata:
  name: cilium
spec:
  clusterSelector:
    matchLabels:
      {}
  releaseName: cilium
  repoURL: https://helm.cilium.io/
  chartName: cilium
  namespace: kube-system
  valuesTemplate: |
    ipam:
      operator:
        clusterPoolIPv4PodCIDRList:
        - ${POD_CIDR}
    cluster:
      id: 0
      name: default
      k8sClientRateLimit:
        qps: "5"
        burst: "10"
---
apiVersion: addons.cluster.x-k8s.io/v1alpha1
kind: HelmChartProxy
metadata:
  name: onlineboutique
spec:
  clusterSelector:
    matchLabels:
      {}
  releaseName: onlineboutique
  repoURL: oci://us-docker.pkg.dev/online-boutique-ci/charts
  chartName: onlineboutique
  namespace: onlineboutique
  valuesTemplate: |
    redis:
      create: true
    serviceAccounts:
      create: true
    networkPolicies:
      create: false
    cartservice:
      database:
        type: redis
    frontend:
      externalService: true
      virtualService:
        create: false
    sidecars:
      create: false
    authorizationPolicies:
      create: true
---
apiVersion: addons.cluster.x-k8s.io/v1alpha1
kind: HelmChartProxy
metadata:
  name: metallb
spec:
  clusterSelector:
    matchLabels:
      {}
  releaseName: metallb
  repoURL: https://metallb.github.io/metallb
  chartName: metallb
  namespace: metallb-system
  valuesTemplate: |

---
apiVersion: addons.cluster.x-k8s.io/v1alpha1
kind: HelmChartProxy
metadata:
  name: metallb-config
spec:
  clusterSelector:
    matchLabels:
      {}
  releaseName: metallb-config
  repoURL: oci://tccr.io/truecharts/
  chartName: metallb-config
  namespace: metallb-system
  valuesTemplate: |
    operator:
      metallb:
        namespace: metallb-system
    ipAddressPools:
      - name: default-pool
        autoAssign: true
        avoidBuggyIPs: true
        addresses:
          - ${LOAD_BALANCER_RANGES}
    L2Advertisements:
      - name: default-l2-advertisement
        addressPools:
          - default-pool
---
apiVersion: addons.cluster.x-k8s.io/v1alpha1
kind: HelmChartProxy
metadata:
  name: external-dns
spec:
  clusterSelector:
    matchLabels:
      {}
  repoURL: oci://registry-1.docker.io/bitnamicharts/
  chartName: external-dns
  releaseName: external-dns
  namespace: external-dns
  valuesTemplate: |
    provider: pdns
    txtOwnerId: poli
    policy: sync
    logLevel: debug
    sources:
      - istio-virtualservice
      - istio-gateway
      - service
      - ingress
    pdns:
      apiUrl: "${PDNS_ZONE}"
      apiPort: 80
      apiKey: ${PDNS_API_KEY}
---
apiVersion: addons.cluster.x-k8s.io/v1alpha1
kind: HelmChartProxy
metadata:
  name: cronjob
spec:
  clusterSelector:
    matchLabels:
      {}
  repoURL: https://sickhub.github.io/charts
  chartName: sickhub
  releaseName: cronjob
  namespace: cronjob
  valuesTemplate: |
    image:
      registry: docker.io
      repository: bitnami/kubectl
      pullPolicy: IfNotPresent
      tag: latest

    jobs:
      patch-frontend:
        schedule: "*/5 * * * *"
        timeZone: "Etc/UTC"
        command: ["/bin/bash", "-c"]
        args:
          - >
            kubectl patch service/frontend-external --namespace=onlineboutique --type=merge --patch='{"metadata": {"annotations": {"external-dns.alpha.kubernetes.io/hostname": "onlineboutique.${CLUSTER_NAME}.ephemeral.mydns.zone"}}}'
    clusterrole:
      create: true
      name: "cron-patch-frontend-svc"
      rules:
      - apiGroups: [""]
        resources: ["services"]
        verbs: ["list", "get", "watch", "patch"]
---
EOF
```

Deploy cluster itself

```sh
clusterctl generate cluster $CLUSTER_NAME -n $CLUSTER_NAME \
  --infrastructure proxmox \
  --kubernetes-version v1.29.6 \
  --control-plane-machine-count 1 \
  --worker-machine-count 1 > ./${CLUSTER_NAME}-cluster.yaml
yq -i 'select(.kind == "Cluster").spec.clusterNetwork.pods.cidrBlocks[] = strenv(POD_CIDR)' ./${CLUSTER_NAME}-cluster.yaml
yq -i 'select(.kind == "Cluster").spec.clusterNetwork += {"services": {"cidrBlocks": [strenv(SVC_CIDR)]}}' ./${CLUSTER_NAME}-cluster.yaml
kubectl apply -n ${CLUSTER_NAME} -f ./${CLUSTER_NAME}-cluster.yaml
```

Once your cluster is provisioned, you can retrieve `kubectl` as shown below and interact with your newly created kubernetes cluster which runs on proxmox:

```sh
❯ clusterctl get kubeconfig ${CLUSTER_NAME} > ~/.kube/${CLUSTER_NAME}
❯ KUBECONFIG=./${CLUSTER_NAME} kubectl get po -A
```
