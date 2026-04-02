# Kubernetes Troubleshooting: Comprehensive Research Report

## Executive Summary

Kubernetes troubleshooting is a critical skill for DevOps engineers, SREs, and platform teams. According to the Spectro Cloud State of Kubernetes 2025 report, IT teams spend an average of **34 working days per year** resolving Kubernetes problems. This report provides a comprehensive guide to diagnosing and resolving common Kubernetes issues across pods, networking, storage, nodes, and security.

---

## 1. Common Kubernetes Issues & Diagnostic Workflow

### The Standard Debugging Workflow

A systematic approach to troubleshooting Kubernetes follows this pattern:

```
get → describe → logs → debug
```

| Step | Command | Purpose |
|------|---------|---------|
| 1 | `kubectl get pods` | Identify problematic pods |
| 2 | `kubectl describe pod <pod>` | View events and configuration |
| 3 | `kubectl logs <pod>` | Examine application logs |
| 4 | `kubectl debug <pod>` | Interactive debugging |

### Essential kubectl Commands for Troubleshooting

```bash
# List all pods with status
kubectl get pods -A

# Get detailed pod information
kubectl get pod <pod-name> -o yaml

# Watch resources in real-time
kubectl get pods -w

# Check events sorted by time
kubectl get events -A --sort-by='.lastTimestamp'

# Check resource usage
kubectl top nodes
kubectl top pods --all-namespaces --sort-by=memory

# Test permissions
kubectl auth can-i list pods --as=system:serviceaccount:default:my-sa
```

---

## 2. Pod Issues & CrashLoopBackOff

### Understanding CrashLoopBackOff

CrashLoopBackOff is one of the most common Kubernetes errors, indicating a container is repeatedly crashing and restarting with exponential backoff (10s, 20s, 40s, up to 5 minutes).

**Pod Lifecycle:**
```
Running → Error/Completed → CrashLoopBackOff
↑___________________________|
(restart with backoff)
```

### Common Exit Codes

| Exit Code | Meaning | Likely Cause |
|-----------|---------|--------------|
| 0 | Success | Container exited unexpectedly |
| 1 | General error | Application startup failure |
| 137 | SIGKILL | OOMKilled (out of memory) |
| 139 | SIGSEGV | Segmentation fault |
| 143 | SIGTERM | Graceful termination |

### Diagnosing CrashLoopBackOff

```bash
# Step 1: Check pod status
kubectl get pods | grep CrashLoopBackOff

# Step 2: Describe the pod for events
kubectl describe pod <pod-name>

# Step 3: Check logs from previous container instance
kubectl logs <pod-name> --previous

# Step 4: Get termination details
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated}'
```

### Common Causes & Solutions

#### 1. Missing Configuration (ConfigMaps/Secrets)
```bash
# Check if ConfigMap exists
kubectl get configmap <config-name>

# Verify mounts
kubectl describe pod <pod-name> | grep -A 10 "Mounts"
```

**Fix:** Create missing ConfigMaps or Secrets
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  KEY: value
```

#### 2. OOMKilled (Out of Memory)
```bash
# Check for OOMKilled status
kubectl describe pod <pod-name> | grep -A 5 "Last State"
```

**Fix:** Increase memory limits
```yaml
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"  # Increase this
```

#### 3. Image Pull Errors (ImagePullBackOff)
```bash
# Check events for image pull failures
kubectl describe pod <pod-name> | grep -A 5 "Events"
```

**Common causes:**
- Incorrect image tag
- Missing image pull secrets
- Private registry authentication issues

### Using kubectl debug for Crashing Pods

```bash
# Create a copy of crashing pod with sleep command
kubectl debug my-crashing-pod -it --copy-to=debug-pod \
  --container=my-container -- sleep infinity

# Then exec into the debug copy
kubectl exec -it debug-pod -c my-container -- /bin/sh

# Test the application manually
./start.sh

# Clean up
kubectl delete pod debug-pod
```

---

## 3. Networking Troubleshooting

### Kubernetes Networking Architecture

```
External User → Load Balancer → Ingress → Service (ClusterIP) → Pods
                                    ↓
                              kube-proxy (iptables/IPVS)
                                    ↓
                              CNI Plugin (Calico/Cilium/Flannel)
```

### Essential Network Debug Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: netshoot
spec:
  containers:
    - name: netshoot
      image: nicolaka/netshoot:latest
      command: ["sleep", "infinity"]
      securityContext:
        capabilities:
          add: [NET_ADMIN, NET_RAW]
```

### DNS Troubleshooting

```bash
# Check DNS resolution
kubectl exec -it netshoot -- nslookup kubernetes.default

# Check /etc/resolv.conf
kubectl exec -it netshoot -- cat /etc/resolv.conf

# Test with dig
kubectl exec -it netshoot -- dig @10.96.0.10 kubernetes.default.svc.cluster.local

# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

### Service Connectivity Issues

```bash
# Check service endpoints
kubectl get endpoints <service-name>

# If empty, check selector matches pod labels
kubectl get pods -l app=my-app
kubectl describe svc <service-name>

# Test service connectivity
kubectl exec -it netshoot -- curl -v http://<service-name>:<port>

# Check kube-proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy
```

### Common Service Problems

| Problem | Symptom | Solution |
|---------|---------|----------|
| No endpoints | Service has no backend pods | Check selector labels match pod labels |
| Port mismatch | Connection refused | Verify targetPort matches container port |
| DNS failure | Name not resolved | Check CoreDNS is running |
| NetworkPolicy blocking | Connection timeout | Review NetworkPolicy rules |

### Network Policy Debugging

```bash
# List all network policies
kubectl get networkpolicies -A

# Test connectivity with netshoot
kubectl exec -it netshoot -- curl -v http://<target-pod>:8080
```

### Packet Capture

```bash
# Capture traffic on pod interface
kubectl exec -it netshoot -- tcpdump -i eth0 -w /tmp/capture.pcap

# Copy capture file locally
kubectl cp netshoot:/tmp/capture.pcap ./capture.pcap
```

---

## 4. Storage & Persistent Volume Issues

### PVC Stuck in Pending

```bash
# Describe PVC to see events
kubectl describe pvc <pvc-name>

# Check for available PVs
kubectl get pv

# Check StorageClass
kubectl get storageclass
```

**Common Causes:**
1. No matching PV for static provisioning
2. Missing StorageClass for dynamic provisioning
3. StorageClass doesn't allow volume expansion

### Volume Mount Failures

```bash
# Check pod events for mount errors
kubectl describe pod <pod-name> | grep -A 5 "FailedMount"

# Check volume attachments
kubectl get volumeattachment

# Check CSI driver status
kubectl get pods -n kube-system | grep csi
```

### PV in Released State

```bash
# Check PV status
kubectl get pv

# If status is "Released", remove claimRef to reuse
kubectl patch pv <pv-name> -p '{"spec":{"claimRef": null}}'
```

### Volume Expansion Issues

```bash
# Check if StorageClass allows expansion
kubectl describe storageclass <sc-name> | grep allowVolumeExpansion

# Enable expansion
allowVolumeExpansion: true
```

---

## 5. Node Issues

### Node Not Ready

```bash
# Check node status
kubectl get nodes

# Describe node for details
kubectl describe node <node-name>

# Check node conditions
kubectl get node <node-name> -o jsonpath='{.status.conditions}'
```

### Common Node Problems

| Issue | Log Indicators | Solution |
|-------|---------------|----------|
| Kubelet down | "KubeletNotReady" | Restart kubelet service |
| Container runtime down | "container runtime is down" | Restart containerd/Docker |
| CNI issues | "no IP addresses available" | Check CNI plugin, IP pool |
| Resource exhaustion | "DiskPressure/MemoryPressure" | Clean up resources |
| TLS timeout | "TLS handshake timeout" | Check network connectivity |

### Debugging Node Issues

```bash
# Debug node directly
kubectl debug node/<node-name> -it --image=ubuntu:22.04

# Inside node debug pod:
chroot /host

# Check kubelet logs
journalctl -u kubelet --since "10 minutes ago"

# Check container runtime
journalctl -u containerd --since "10 minutes ago"

# Check system resources
top
free -h
df -h
```

### PID Exhaustion

```bash
# Monitor thread count per cgroup
ps -e -w -o "thcount,cgname" --no-headers | \
  awk '{a[$2] += $1} END{for (i in a) print a[i], i}' | \
  sort --numeric-sort --reverse | head -8
```

---

## 6. RBAC & Permission Issues

### Understanding RBAC Errors

```
Error from server (Forbidden): pods is forbidden: 
User "system:serviceaccount:production:app-sa" cannot list resource "pods" 
in API group "" in the namespace "production"
```

### Diagnosing RBAC Issues

```bash
# Test permissions directly
kubectl auth can-i list pods --as=system:serviceaccount:production:app-sa -n production

# List all permissions for a service account
kubectl auth can-i --list --as=system:serviceaccount:production:app-sa -n production

# Find role bindings for service account
kubectl get rolebindings -n production -o json | \
  jq -r '.items[] | select(.subjects[]?.name=="app-sa") | .metadata.name'
```

### Common RBAC Fixes

```yaml
# Create a Role with specific permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
# Bind the role to service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-sa-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 7. kubectl debug Command Reference

### Ephemeral Debug Containers (K8s 1.25+)

```bash
# Attach debug container to running pod
kubectl debug -it my-pod --image=busybox --target=my-container

# Use netshoot for network debugging
kubectl debug -it my-pod --image=nicolaka/netshoot --target=my-container

# Specify custom container name
kubectl debug -it my-pod --image=busybox --container=debugger --target=my-container
```

### Copy Pod for Debugging

```bash
# Copy pod with overridden command
kubectl debug my-crashing-pod -it --copy-to=debug-pod \
  --container=my-container -- /bin/sh

# Copy with different image
kubectl debug my-pod -it --copy-to=debug-pod \
  --set-image=my-container=ubuntu:22.04
```

### Node Debugging

```bash
# Create debug pod on specific node
kubectl debug node/my-node -it --image=ubuntu:22.04

# Access host filesystem
chroot /host
```

### What You Can Do Inside Debug Containers

```bash
# Network diagnostics
ping google.com
curl -v http://my-service:8080/healthz
nslookup my-service.my-namespace.svc.cluster.local
tcpdump -i eth0 -n port 8080

# Filesystem inspection
ls /proc/1/root/app/
cat /proc/1/environ | tr '\0' '\n'

# Check resource limits
cat /sys/fs/cgroup/memory.max
cat /sys/fs/cgroup/cpu.max
```

---

## 8. Observability & Monitoring Tools

### The Three Pillars of Observability

| Pillar | Tool Examples | Purpose |
|--------|---------------|---------|
| **Metrics** | Prometheus, Grafana | Quantitative performance data |
| **Logs** | ELK/EFK, Loki, Fluentd | Event-level insights |
| **Traces** | Jaeger, OpenTelemetry | Request flow across services |

### Top Observability Tools for Kubernetes (2025)

#### 1. Prometheus + Grafana (Open Source Stack)
- **Prometheus**: Time-series metrics collection with PromQL
- **Grafana**: Visualization and dashboards
- **Best for**: Self-managed, cost-effective monitoring

#### 2. Jaeger (Distributed Tracing)
- End-to-end request visibility
- Service dependency mapping
- OpenTracing compliant

#### 3. OpenTelemetry
- Vendor-neutral instrumentation
- Collects metrics, logs, and traces
- Native Kubernetes integration

#### 4. Commercial Platforms
- **Datadog**: Full-stack monitoring with K8s-specific dashboards
- **New Relic**: APM with distributed tracing
- **Dynatrace**: AI-powered monitoring and automation
- **Splunk**: Log aggregation and analytics

### Essential Metrics to Monitor

**Cluster Level:**
- Node CPU/Memory usage
- Disk pressure
- API server latency
- etcd health

**Pod Level:**
- CPU/Memory requests vs limits
- Restart counts
- Network I/O
- Error rates

**Application Level:**
- Request throughput
- Response times
- Error rates
- Custom business metrics

---

## 9. Best Practices for Troubleshooting

### Prevention Strategies

1. **Add Health Checks Early**
   - Liveness probes: Detect deadlocks
   - Readiness probes: Control traffic flow
   - Startup probes: Handle slow-starting apps

2. **Set Resource Limits**
   ```yaml
   resources:
     requests:
       memory: "256Mi"
       cpu: "250m"
     limits:
       memory: "512Mi"
       cpu: "500m"
   ```

3. **Use Init Containers for Dependencies**
   - Don't assume services are available
   - Wait for databases, message queues

4. **Centralized Logging**
   - Aggregate logs from all pods
   - Use structured logging (JSON)
   - Define retention policies

5. **Version Control All Manifests**
   - Store YAML in Git
   - Use GitOps workflows
   - Enable easy rollbacks

### Production Readiness Checklist

- [ ] Resource requests and limits defined
- [ ] Health probes configured
- [ ] Monitoring and alerting in place
- [ ] Centralized logging configured
- [ ] Network policies defined
- [ ] RBAC permissions minimal
- [ ] Secrets management implemented
- [ ] Backup strategy documented
- [ ] Runbook for common issues
- [ ] Disaster recovery tested

### Useful Aliases & Shortcuts

```bash
# Add to ~/.bashrc or ~/.zshrc
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias kgp='kubectl get pods'
alias kgn='kubectl get nodes'
alias kdf='kubectl delete -f'
alias kaf='kubectl apply -f'

# Autocomplete
source <(kubectl completion bash)
```

---

## 10. Quick Reference: Diagnostic Scripts

### CrashLoopBackOff Debug Script

```bash
#!/bin/bash
POD=$1
NAMESPACE=${2:-default}

echo "=== Pod Status ==="
kubectl get pod $POD -n $NAMESPACE

echo -e "\n=== Last Termination State ==="
kubectl get pod $POD -n $NAMESPACE -o jsonpath='{.status.containerStatuses[0].lastState.terminated}' | jq .

echo -e "\n=== Previous Logs ==="
kubectl logs $POD -n $NAMESPACE --previous --tail=50

echo -e "\n=== Recent Events ==="
kubectl get events -n $NAMESPACE --field-selector involvedObject.name=$POD --sort-by='.lastTimestamp'
```

### Network Debug Script

```bash
#!/bin/bash
POD_NAME=$1
NAMESPACE=${2:-default}

echo "=== Pod Network Info ==="
kubectl exec -n $NAMESPACE $POD_NAME -- ip addr show
kubectl exec -n $NAMESPACE $POD_NAME -- ip route show

echo "=== DNS Resolution ==="
kubectl exec -n $NAMESPACE $POD_NAME -- cat /etc/resolv.conf
kubectl exec -n $NAMESPACE $POD_NAME -- nslookup kubernetes.default

echo "=== External Connectivity ==="
kubectl exec -n $NAMESPACE $POD_NAME -- curl -s -o /dev/null -w "%{http_code}" https://google.com
```

### Storage Debug Script

```bash
#!/bin/bash
PVC_NAME=$1
NAMESPACE=${2:-default}

echo "=== PVC Status ==="
kubectl get pvc $PVC_NAME -n $NAMESPACE
kubectl describe pvc $PVC_NAME -n $NAMESPACE

PV_NAME=$(kubectl get pvc $PVC_NAME -n $NAMESPACE -o jsonpath='{.spec.volumeName}')
if [ -n "$PV_NAME" ]; then
  echo -e "\n=== PV Status ==="
  kubectl get pv $PV_NAME
  kubectl describe pv $PV_NAME
  
  echo -e "\n=== Volume Attachments ==="
  kubectl get volumeattachment | grep $PV_NAME
fi
```

---

## Summary

Effective Kubernetes troubleshooting requires:

1. **Systematic Approach**: Follow the `get → describe → logs → debug` workflow
2. **Right Tools**: Use kubectl debug, netshoot, and observability platforms
3. **Understanding Layers**: Know how CNI, Services, DNS, and storage interact
4. **Check Basics First**: Pod IPs, endpoints, DNS resolution
5. **Use Logs**: Application, kubelet, CNI, and CoreDNS logs
6. **Prevent Issues**: Set resource limits, health probes, and monitoring early

---

## References

- [Kubernetes Official Documentation](https://kubernetes.io/docs/)
- [kubectl Debug Guide - OneUptime](https://oneuptime.com/blog/post/2026-02-20-kubernetes-kubectl-debug-guide/view)
- [CrashLoopBackOff Troubleshooting - SFEIR Institute](https://institute.sfeir.com/en/kubernetes-training/debug-pod-crashloopbackoff-causes-solutions-kubernetes/)
- [Kubernetes Networking Deep Dive](https://medium.com/@h.stoychev87/kubernetes-networking-a-deep-dive-6081d794e97c)
- [CNCF Top Kubernetes Troubleshooting Techniques](https://www.cncf.io/blog/2025/09/12/top-kubernetes-k8s-troubleshooting-techniques-part-1/)
