### Pod
#### Definition
- Pod is minimum unit in k8s
- Container is always run in Pod
#### Purpose
- Wrap one or more containers that need to run together
- Give containers a shared network + storage context
#### Architecture
Pod -> Container(s)
- kubelet on the node runs Pod via container runtime (containerd)
#### Key Concepts
- Shared resources: containers in same pod
	- same IP
	- same network namespace (talk via localhost)
	- share volume
- Pod can contain
	- one or more container (main + sidecar / init container)
- Life cycle
	- Container crash -> Kubelet restart -> CrashLoopBackOff
	- `restartPolicy`: Always (default) | OnFailure | Never
- Limitation: Pod is not support (That's why not create Pod directly)
	- Self healing
	- Scaling
	- Rolling update
#### Commands
```bash
k run nginx --image=nginx
k run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
k get po -o wide
k describe po <pod-name>
k logs <pod-name> [-c <container>] [--previous]
k exec -it <pod-name> -- sh
k delete po <pod-name>
```

### Init Container & Sidecar
#### Definition
#### Purpose
#### Architecture
#### Key Concepts
#### Commands

### Probes
#### Definition
#### Purpose
#### Architecture
#### Key Concepts
#### Commands

### Resources (Requests / Limits) & QoS
#### Definition
#### Purpose
#### Architecture
#### Key Concepts
#### Commands

### Static Pod
#### Definition
#### Purpose
#### Architecture
#### Key Concepts
#### Commands

### Labels, Selectors & Namespaces
#### Definition
#### Purpose
#### Architecture
#### Key Concepts
#### Commands

### ReplicaSet
#### Definition
- ReplicaSet ensure number of Pod running
#### Purpose
- Self healing: Pod die -> ReplicaSet create new Pod
- Scaling: change number of replicas
#### Architecture
ReplicaSet -> Pod
- ReplicaSet finds its Pods by label selector
#### Key Concepts
- Main fields
```yaml
spec:
  replicas: 3        # desired number of Pods
  selector:          # which Pods belong to this ReplicaSet
    matchLabels:
      app: web
  template:          # Pod template (labels must match selector)
```
- Limitation: ReplicaSet not support
	- version management
	- rolling Update
	- rollBack
	- -> Use Deployment instead (don't create ReplicaSet directly)
#### Commands
```bash
k get rs
k describe rs <rs-name>
k scale rs <rs-name> --replicas=3
k delete rs <rs-name>
```

### Deployment
#### Definition
- Deployment manages ReplicaSets to provide declarative updates for Pods
#### Purpose
- Provide
	- Rolling update
	- Rollback
	- Scaling
	- Version management
#### Architecture
Deployment -> ReplicaSet -> Pod
#### Key Concepts
- Rolling update
	- Deployment tạo ReplicaSet mới
	- ↓
	- Scale Up ReplicaSet mới
	- ↓
	- Scale Down ReplicaSet cũ
- Strategy
```yaml
spec:
  strategy:
    type: RollingUpdate   # default | Recreate (kill all old Pods first)
    rollingUpdate:
      maxSurge: 25%       # extra Pods allowed above replicas
      maxUnavailable: 25% # Pods allowed to be down during update
```
- Deployment revision
	- Only create a new revision of deployment when pod template changes
	```yaml
	spec:
	  template:
	```
	- Scale (change `replicas`) does NOT create new revision
#### Commands
```bash
k create deploy nginx --image=nginx --replicas=2
k get deploy
k describe deploy <deployment>
k edit deploy <deployment>
k set image deploy/<deployment> <container>=nginx:1.25
k rollout status deploy <deployment>
k rollout history deploy <deployment>
k rollout undo deploy <deployment> [--to-revision=1]
k scale deploy <deployment> --replicas=2
```

### StatefulSet
#### Definition
- StatefulSet is a workload resource for stateful applications.
#### Purpose
- Provides:
	- Stable Identity
	- Stable storage
	- Ordered Deployment
	- Ordered Scaling
#### Architecture
StatefulSet -> Pods (+ Headless Service + PVC per Pod)
Each Pod has:
- Stable name
- Stable DNS (via headless Service)
- Stable PVC
#### Key Concepts
- Stable Identity
```text
mysql-0
mysql-1
mysql-2
```
- Headless Service (required, `spec.serviceName`)
```yaml
# Service with clusterIP: None -> each Pod gets its own DNS record:
# <pod>.<svc>.<ns>.svc.cluster.local  e.g. mysql-0.mysql.default.svc.cluster.local
spec:
  serviceName: mysql
```
- Stable Storage
```text
volumeClaimTemplates (name: data)
mysql-0          mysql-1
↓                ↓
data-mysql-0     data-mysql-1     # PVC name = <template>-<pod>
```
	- PVCs are NOT deleted when scale down / delete StatefulSet
- Ordered Scaling
```text
Scale up: 0 -> 1 -> 2  Scale Down: 2 -> 1 -> 0
```
	- `podManagementPolicy: Parallel` -> skip ordering (default `OrderedReady`)
#### Commands
```bash
k get sts
k describe sts <sts-name>
k scale sts <sts-name> --replicas=<n>
```
### DaemonSet
#### Definition
- Ensure that every node has a running pod
#### Purpose
- Run node-level agent on every node
- Typical Use Cases
	- kube-proxy
	- aws-node
	- node-exporter
	- nvidia-device-plugin
#### Architecture
DaemonSet -> Pod (one per Node)
- New node join -> Pod auto created on that node
- Node removed -> Pod garbage collected
#### Key Concepts
- Eligible node count = Pod count (nodes with untolerated taints get no Pod)
- No replicas
- Run on control plane node -> need toleration for its taint
- Run on subset of nodes -> `nodeSelector` / node affinity
#### Commands
```bash
k get ds -A
k describe ds <ds-name> -n <namespace>
k rollout status ds <ds-name>
# No "k create ds" -> generate Deployment yaml, then:
#   kind: Deployment -> DaemonSet, remove replicas + strategy
k create deploy <name> --image=<image> --dry-run=client -o yaml > ds.yaml
```

### Job
#### Definition
- A Job is a workload resource that runs a finite task and terminates after the task is completed successfully.
#### Purpose
- Run one-time or batch workloads.
- Typical usecases:
	- Database backup
	- Data import
	- Data export
	- Report generation
#### Architecture
Job -> Pod
- A Job create one or more Pods to complete a task.
#### Key Concepts
- Completions
```yaml
# Define how many successful executions are required
completions:    # default 1
```
- Retry
```yaml
# Container exit with error (Exit code != 0)
# Number of retries is controlled by:
backoffLimit:   # default 6
```
- Parallelism
```yaml
# Run multiple Pods simultaneously:
parallelism:
```
- restartPolicy must be `Never` or `OnFailure` (not Always)
- Time limit / cleanup
```yaml
activeDeadlineSeconds: 100      # kill Job if running longer than this
ttlSecondsAfterFinished: 60     # auto delete Job after finished
```
#### Commands
```bash
k create job <job-name> --image=busybox -- echo hello
k get jobs
k describe job <job-name>
k logs job/<job-name>
k delete job <job-name>
```

### CronJob
#### Definition
- A CronJob creates Jobs on a scheduled basis.
#### Purpose
- Run recurring tasks automatically.
- Typical use cases:
	- Daily Database Backup
	- Log Cleanup
	- Report
	- Scheduled task
#### Architecture
CronJob -> Job -> Pod
#### Key Concepts
- Schedule
```yaml
# Uses standard cron syntax: min hour day-of-month month day-of-week
schedule: "0 0 * * *"  # Runs everyday at midnight.
```
- Failure
	- If a Job fails, the next scheduled execution is unaffected.
- Concurrency
```yaml
concurrencyPolicy: Allow   # default | Forbid (skip new run if old still running) | Replace (kill old, run new)
suspend: true              # pause scheduling
```
- History
```yaml
successfulJobsHistoryLimit: 3   # default
failedJobsHistoryLimit: 1       # default
```
#### Commands
```bash
k get cronjobs
k describe cronjob <cronjob-name>
k create cronjob hello \
      --image=busybox \
      --schedule="*/5 * * * *" \
      -- echo hello
# Trigger a run immediately (test the CronJob)
k create job test-run --from=cronjob/<cronjob-name>
k delete cronjob <cronjob-name>
```
### HPA
#### Definition
- HPA is kubernetes controller to automatic increase or decrease number of pods based on metrics as (CPU, memory or custom metrics)
#### Purpose
- Automatic scale out/in on demand
#### Architecture
- Pods -> Metrics Server -> HPA -> Deployment -> ReplicaSet -> Pods
- Requires metrics-server installed (otherwise `k top` fails, HPA target `<unknown>`)
#### Key Concepts
- Horizontal Scale (more Pods), not Vertical (bigger Pods)
- Utilization % is calculated against **requests** (not limits)
	- No `resources.requests` set -> HPA cannot compute -> target `<unknown>`
- Formula
```text
desiredReplicas = ceil(currentReplicas * currentMetric / targetMetric)
e.g. 2 Pods, CPU 160%, target 80% -> ceil(2 * 160/80) = 4
```
#### Commands
```bash
k get hpa
k top pod
k top node
k autoscale deployment nginx \
		--cpu-percent=80 \
		--min=2 \
		--max=5
```

### PodDisruptionBudget (PDB)
#### Definition
- PDB is a kubernetes resource that used to limit number of pod can be disrupted in voluntary disruptions.
#### Purpose
- PDB help to keep application available when:
	- Node drain
	- Cluster upgrade
	- Node maintenance
	- Pod eviction
#### Architecture
PDB --(label selector)--> Pods
- `k drain` -> Eviction API -> check PDB -> allow / reject (drain waits & retries)
#### Key Concepts
- Voluntary disruption (checked by PDB, via Eviction API)
```bash
k drain <node> --ignore-daemonsets
# cluster autoscaler scale down, node upgrade
```
- NOT checked by PDB (bypass)
	- `k delete pod` (direct delete, no eviction)
	- `k delete deploy`
	- Deployment rolling update / `k rollout restart` -> controlled by `maxSurge` / `maxUnavailable` of Deployment
- Involuntary disruption (PDB cannot prevent)
	- Node crash
	- Network failure
- minAvailable vs maxUnavailable (set only ONE of them, number or %)
```yaml
spec:
  minAvailable: 2        # at least 2 Pods must stay running
  # maxUnavailable: 1    # at most 1 Pod can be down at a time
  selector:
    matchLabels:
      app: web
```
#### Commands
```bash
k get pdb
k describe pdb <pdb-name>   # check "Allowed disruptions"
k create pdb web-pdb --selector=app=web --min-available=2
k create pdb web-pdb --selector=app=web --max-unavailable=1
```