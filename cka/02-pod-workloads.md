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
- Node count = Pod count
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
completions:
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

### Workload Comparison
