### Control Plane
#### Definition
#### Purpose
#### Architecture
#### Key Concepts
- How components run (kubeadm)
	- API Server, Scheduler, Controller-manager, Etcd (stacked) -> Static Pod (see 02 Static Pod)
	- coredns -> Deployment
	- kube-proxy -> DaemonSet because every node needs kube-proxy
#### Commands
Check log when kubectl is connection refused (API server down -> kubectl not work)
```bash
# check container log
crictl ps -a | grep kube-
crictl logs <container-id>
```

### API Server
#### Definition
#### Purpose
#### Architecture
#### Key Concepts
#### Commands

### Scheduler
#### Definition
#### Purpose
#### Architecture
#### Key Concepts
#### Commands

### Controller Manager
#### Definition
#### Purpose
#### Architecture
#### Key Concepts
#### Commands

### ETCD
#### Definition
- distributed key-value store
- store resource configuration in k8s
#### Purpose
#### Architecture
- kubeadm stacked etcd: run as Static Pod
#### Key Concepts
#### Commands

### Kubelet
#### Definition
- responsible to run pod
#### Purpose
#### Architecture
#### Key Concepts
#### Commands
Check log kubelet
```bash
systemctl status kubelet
journalctl -u kubelet -f
systemctl restart kubelet
```

### kube-proxy
#### Definition
- Network agent that runs on every node, implements Service.
#### Purpose
- Forward traffic that was sent to ClusterIP (Service IP) to one of the Pod IPs
#### Architecture
- Runs as DaemonSet (1 Pod per node, name has random suffix e.g. `kube-proxy-x7k2p`)
- Watch Service + EndpointSlice -> write rule iptables/IPVS -> kernel DNAT
- iptables chain: `KUBE-SERVICES -> KUBE-SVC-xxx (1 per Service) -> KUBE-SEP-xxx (1 per Pod, DNAT)`
	- Pod is chosen randomly (probability)
#### Key Concepts
- kube-proxy does NOT forward packets, the kernel does that
- ClusterIP is a virtual IP, not bound to any network interface (iptables / nftables mode)
	- IPVS mode: bound to dummy interface `kube-ipvs0`, still virtual & not reachable from outside
- DNAT happens on the node of the sending Pod
	- DNAT modifies destination IP:port (DNAT != DNS, DNS = name -> IP)
- Mode: same job (ClusterIP -> DNAT -> Pod IP), different kernel tool

| Mode     | Lookup                                           | Update              | Load balancing | Status                     |
| -------- | ------------------------------------------------ | ------------------- | -------------- | -------------------------- |
| iptables | rule list, top -> down (slow with many Services) | rewrite whole table | random only    | default                    |
| ipvs     | hash table (fast)                                | incremental         | rr, lc, sh...  | being replaced by nftables |
| nftables | map / set (fast)                                 | incremental         | random         | GA 1.33, future            |

- Change mode: edit ConfigMap -> must restart DaemonSet (kube-proxy reads config only at startup)
- Only Ready Pods are in EndpointSlice
- Service doesn't run on any node -> ClusterIP is reachable from every node & every Pod, NOT from outside the cluster
- kube-proxy down: old rules still exist, new Service or Pod changes will not be updated on that node.
#### Commands
```bash
k get ds kube-proxy -n kube-system
k get cm kube-proxy -n kube-system -o yaml | grep mode   # mode: "" -> iptables
k logs -n kube-system <kube-proxy-pod> | grep -i proxier # "Using iptables Proxier"
# Show rules on node
iptables -t nat -L KUBE-SERVICES -n | grep <ClusterIP>   # iptables
ipvsadm -Ln                                              # ipvs
nft list table ip kube-proxy                             # nftables
# Change mode
k edit cm kube-proxy -n kube-system                      # mode: "ipvs"
k rollout restart ds kube-proxy -n kube-system
```

### Container Runtime
#### Definition
#### Purpose
#### Architecture
#### Key Concepts
#### Commands

### Request Flow
- Kubectl -> API Server -> Etcd -> Controllers -> Scheduler -> Kubelet (every step goes through API Server)
	- API Server (Authentication -> Authorization -> Admission (mutate, validate)) -> save to Etcd
	- Deployment Controller: create ReplicaSet
	- ReplicaSet Controller: create Pod (Pending)
	- Scheduler: find node -> set `spec.nodeName` (Binding)
	- kubelet: watch API Server -> detect a new pod -> CRI -> containerd create container, CNI assign Pod IP
	- kubelet call API Server -> update pod status Running
- Rule:
	- Components communicate with each other by watch API Server, not call directly
	- Only API Server talks to Etcd
