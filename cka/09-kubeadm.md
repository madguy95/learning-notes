### Cluster Initialization

### Node Join

### Token Management

### Certificate Management

### Node Maintenance (cordon / drain / uncordon)

### Cluster Upgrade

### High Availability (HA) Control Plane
- Each control plane node has its own manifest folder. If there's any change, need to change it on every node.
- API Server: all pods work at the same time behind the load balancer.
- Scheduler, Controller-manager: use leader election, only 1 pod is active. If the leader goes down, another one will be promoted (after Lease expires, ~15s).
```bash
k get lease -n kube-system   # HOLDER column = current leader
```
- Etcd: need quorum (floor(n/2) + 1).
	- quorum: a decision needs more than half of the total members to agree.
	- with total = 3, when 1 node down -> 2 nodes running -> still works.
	- with total = 5, when 3 nodes down -> 2 nodes running -> violates quorum -> stops.
	- quorum is counted on total members, not on running members.
	- Use odd numbers (3, 5): even numbers don't tolerate more failures (4 nodes still tolerate only 1).
	- "More than half" avoids split-brain: when the network is split, only one side can write.
- Etcd lost quorum -> API Server almost hangs (both get and create).
	- Running Pods on workers still run, crashed containers are still restarted by kubelet (restartPolicy).
	- Lost: create / update / scale, schedule new Pods, recreate Pods of a dead node on another node.

### Common Commands
