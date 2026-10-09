## General flow
Apply to every issue, in order:
```bash
k get <resource> -o wide                    # 1. status overview (STATUS / READY / RESTARTS)
k describe <resource> <name>                # 2. Events at the bottom
k logs <pod> [-c <container>] [--previous]  # 3. app log (--previous = crashed container)
k exec -it <pod> -- sh                      # 4. check from inside container
k get events -n <ns> --sort-by=.lastTimestamp  # 5. recent events of namespace
```
- Structure of each case: **Symptom → Keyword → Diagnose → Cause → Fix**

## Pod
### 1. Pending
**Symptom**
- STATUS = Pending, NODE = `<none>`

**Keyword**
```text
- Insufficient CPU
- Insufficient memory
- didn't match Pod's node affinity/selector
- had taint that the pod didn't tolerate
- pod has unbound immediate PersistentVolumeClaims
```

**Diagnose**
```bash
k describe pod <pod>              # Events
k get node --show-labels
k describe node <node> | grep -i taint
k describe node <node> | grep -A 8 "Allocated resources"
```

**Cause → Fix**
- Resource (Insufficient CPU/Memory) → reduce `resources.requests` (CPU/Memory)
  ```bash
  k edit deploy <deployment>
  ```
- PVC not bound → see [[#PVC Pending]]
  ```bash
  k describe pvc <pvc>
  ## check pv, StorageClass, Capacity, AccessMode
  k get pv,sc
  ```
- Node label (nodeSelector / affinity not match) → add label to node, or edit affinity in pod
  ```bash
  k label node <node> key=value
  ```
- Taint/Toleration → add toleration to pod, or remove taint from node
  ```bash
  k edit deploy <deployment>       # spec.template.spec.tolerations
  k taint node <node> key=value:NoSchedule-   # remove taint
  ```
- No Events at all → kube-scheduler is down → see [[#Control plane component down]]

### 2. ContainerCreating
**Symptom**
- STATUS = ContainerCreating for a long time (Pod already scheduled to a node)

**Keyword**
```text
FailedMount
MountVolume.SetUp failed
AttachVolume failed
volume not found
configmap "xxx" not found
secret "xxx" not found
failed to create pod sandbox / network plugin not ready
```

**Diagnose**
```bash
k describe pod <pod>
k get pod <pod> -o yaml
## check volumeMounts, volumes, persistentVolumeClaim, configMap, secret
k get cm,secret,pvc
```

**Cause → Fix**
- Volume mount failed / PVC issue → fix PVC name in `volumes`, check PVC is Bound
- ConfigMap/Secret volume not found → create missing ConfigMap/Secret, or fix name in `volumes`
- CNI not ready (sandbox error) → check CNI pods in kube-system
- Note: ContainerCreating only relates to **volume** section. ConfigMap/Secret used as `env` (not volume) does not cause this state → it causes `CreateContainerConfigError`

### 3. ImagePullBackOff / ErrImagePull
**Symptom**
- STATUS = ImagePullBackOff / ErrImagePull

**Keyword**
```text
- Failed to pull image "nginxx:latest": manifest unknown / not found
- Unauthorized
- authentication required
```

**Diagnose**
```bash
k describe pod <pod>
k get pod <pod> -o jsonpath='{.spec.containers[*].image}'
```

**Cause → Fix**
- Wrong image/tag → fix image
  ```bash
  k set image deploy/<deployment> <container>=nginx:1.25
  # or
  k edit deploy <deployment>
  ```
- Registry credential (private registry) → create secret + add `imagePullSecrets`
  ```bash
  k get secret
  k describe deploy <deployment>   ## check imagePullSecrets
  ```

### 4. CrashLoopBackOff
**Symptom**
- STATUS = CrashLoopBackOff, RESTARTS increasing

**Keyword**
```text
Last State: Terminated
Reason: Error / OOMKilled
Exit Code != 0
```

**Diagnose**
```bash
k describe pod <pod>          # Last State, Reason, Exit Code
k logs <pod> --previous       # log of crashed container
```

**Cause → Fix**
- App error (Exit Code 1) → read log, fix `command` / `args` / `env` / ConfigMap
- OOMKilled (Exit Code 137) → app running out of memory → increase `resources.limits.memory`
  ```bash
  k edit deploy <deployment>
  ```
- Permission → fix `securityContext` (runAsUser, runAsGroup, fsGroup) / volumeMounts
  ```bash
  k get deploy <deployment> -o yaml | grep securityContext -A 20
  ## inside container: check current user, group, folder permission
  id
  ls -ld <folder-path>
  ```

### 5. Running + Ready = 0/1
**Symptom**
- STATUS = Running, READY = 0/1, RESTARTS not increasing

**Keyword**
```text
Readiness probe failed
HTTP probe failed with statuscode: 503
connection refused
```

**Diagnose**
```bash
k describe pod <pod>                      # check readinessProbe config + Events
k exec -it <pod> -- sh
curl localhost:<port>/<health-check>
```

**Cause → Fix**
- Wrong readiness probe config (port / path) → fix `readinessProbe`
- Health endpoint error → fix app / endpoint
- Application is not ready yet → increase `initialDelaySeconds`
- Note: Service will not route traffic to pod (not ready) → Endpoints empty

### 6. Running + Restart Count increasing
**Symptom**
- STATUS = Running (sometimes), RESTARTS increasing

**Keyword**
```text
Liveness probe failed
Startup probe failed
Reason: OOMKilled
```

**Diagnose**
```bash
k describe pod <pod>         # check Last State, Reason, Events
k logs <pod> --previous
```

**Cause → Fix**
- Liveness/Startup probe error (wrong port / path, too strict) → fix probe, increase `initialDelaySeconds` / `failureThreshold`
- Application crash → read log
- OOMKilled → increase `resources.limits.memory`

## SVC
### 1. Endpoints = `<none>`
**Symptom**
- Service exists but `ENDPOINTS <none>`

**Keyword**
```text
ENDPOINTS <none>
```

**Diagnose**
```bash
k get endpoints <svc>
k get svc <svc> -o yaml        # check selector
k get pod --show-labels
```

**Cause → Fix**
- SVC selector not match pod label → fix `spec.selector` in svc (or pod labels)
- Pod is not ready → see [[#5. Running + Ready = 0/1]]

### 2. Endpoints exist but Service not work
**Symptom**
- Endpoints have IP, but call to service fails

**Keyword**
```text
Connection refused
```

**Diagnose**
```bash
k get endpoints <svc>
k describe svc <svc>           # Port, TargetPort
k describe pod <pod>           # containerPort
k run tmp --rm -it --image=busybox -- wget -qO- http://<svc>:<port>
```

**Cause → Fix**
- Wrong port / targetPort → `targetPort` must match `containerPort`

## DNS
### DNS Resolution failed
**Symptom**
- Call by service name fails, call by IP works

**Keyword**
```text
Could not resolve host
server can't find
```

**Diagnose**
```bash
k run tmp --rm -it --image=busybox:1.28 -- nslookup <svc>
k get pod -n kube-system -l k8s-app=kube-dns
k get svc -n kube-system kube-dns
k logs -n kube-system -l k8s-app=kube-dns
```

**Cause → Fix**
- CoreDNS failure (pods not running / crash) → check CoreDNS pods, ConfigMap `coredns` in kube-system
- Wrong name → cross namespace use `<svc>.<namespace>` or `<svc>.<namespace>.svc.cluster.local`

## Storage
### PVC Pending
**Symptom**
- PVC STATUS = Pending

**Keyword**
```text
pod has unbound immediate PersistentVolumeClaims
no persistent volumes available for this claim
```

**Diagnose**
```bash
k describe pvc <pvc>
k get pv
k get sc
```

**Cause → Fix** (PVC and PV must match all) → fix PVC or create matching PV
- No PV
- StorageClass mismatch
- Capacity mismatch (PV capacity < PVC request)
- AccessMode mismatch
	- RWO: one node read/write
	- RWX: many node read/write
	- ROX: many node read only
	- RWOP: one pod read/write
- Note: StorageClass with `volumeBindingMode: WaitForFirstConsumer` → PVC Pending is normal until a Pod uses it

### Permission Denied
**Symptom**
- App cannot read/write file in volume

**Keyword**
```text
Permission denied
Operation not permitted
```

**Diagnose**
```bash
k exec -it <pod> -- sh
# check current user, group
id
# check folder permission
ls -ld <folder-path>
```

**Cause → Fix** → check yaml `securityContext`
- User has no permission → `runAsUser` (Process User Id)
- Wrong group → `runAsGroup` (Process Group Id)
- Missing fsGroup / volume owner → `fsGroup` (Volume Group Id)

## Node
### Node NotReady
**Symptom**
- `k get node` → STATUS = NotReady

**Keyword**
```text
Kubelet stopped posting node status
container runtime network not ready
container runtime is down
```

**Diagnose**
```bash
k describe node <node>                 # Conditions
ssh <node>
sudo systemctl status kubelet
sudo journalctl -u kubelet -f          # kubelet log
sudo systemctl status containerd
```

**Cause → Fix**
- kubelet stopped / disabled → start + enable kubelet
  ```bash
  sudo systemctl enable --now kubelet
  ```
- kubelet config wrong (wrong path, cert, flag) → fix config, then restart
  ```bash
  # config files
  /var/lib/kubelet/config.yaml
  /etc/kubernetes/kubelet.conf
  /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
  sudo systemctl daemon-reload && sudo systemctl restart kubelet
  ```
- Container runtime down → `sudo systemctl restart containerd`
- CNI not ready → check CNI pods in kube-system

## Control plane
### Control plane component down
**Symptom**
- kube-apiserver down → `kubectl` fails: `The connection to the server ... was refused`
- kube-scheduler down → new Pod Pending, no Events, NODE = `<none>`
- kube-controller-manager down → Deployment created but no ReplicaSet/Pod, scale not work

**Keyword**
```text
The connection to the server <ip>:6443 was refused
CrashLoopBackOff (kube-system pods)
```

**Diagnose**
```bash
k get pod -n kube-system
k describe pod -n kube-system <component-pod>
k logs -n kube-system <component-pod>
# apiserver down -> kubectl not work -> use crictl on control plane node
sudo crictl ps -a | grep kube
sudo crictl logs <container-id>
ls /var/log/pods/
```

**Cause → Fix**
- Control plane components are **static pods** → manifest in `/etc/kubernetes/manifests/`
  - kube-apiserver.yaml, kube-scheduler.yaml, kube-controller-manager.yaml, etcd.yaml
- Wrong flag / cert path / etcd endpoint / port in manifest → fix the yaml file
- kubelet auto recreates static pod after file saved (wait ~30s-1m)
- Note: static pod is managed by kubelet → `k edit` / `k delete` does not fix it, must edit manifest file
