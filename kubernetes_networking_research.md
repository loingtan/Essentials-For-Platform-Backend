# Kubernetes Networking: A Comprehensive Research Guide

## Table of Contents
1. [Introduction to Kubernetes Networking](#1-introduction-to-kubernetes-networking)
2. [Container Network Interface (CNI)](#2-container-network-interface-cni)
3. [Kubernetes Service Types](#3-kubernetes-service-types)
4. [Ingress Controllers](#4-ingress-controllers)
5. [Network Policies](#5-network-policies)
6. [DNS and Service Discovery](#6-dns-and-service-discovery)
7. [kube-proxy and Load Balancing](#7-kube-proxy-and-load-balancing)
8. [Popular CNI Plugins Comparison](#8-popular-cni-plugins-comparison)
9. [Best Practices](#9-best-practices)

---

## 1. Introduction to Kubernetes Networking

Kubernetes networking is fundamentally different from traditional Docker networking. While Docker uses host-port mapping, Kubernetes employs a **flat network structure** where every pod gets its own IP address and can communicate directly with other pods without NAT (Network Address Translation).

### Key Networking Requirements in Kubernetes

| Requirement | Description |
|-------------|-------------|
| **Pod-to-Pod Communication** | All pods can communicate with all other pods without NAT |
| **Pod-to-Service Communication** | Pods can reach services via stable virtual IPs |
| **External-to-Service Communication** | External clients can access services through various mechanisms |

### Core Networking Components

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │   Node 1    │    │   Node 2    │    │   Node N    │         │
│  │ ┌─────────┐ │    │ ┌─────────┐ │    │ ┌─────────┐ │         │
│  │ │  Pod A  │ │◄──►│ │  Pod B  │ │◄──►│ │  Pod C  │ │         │
│  │ │10.0.1.5 │ │    │ │10.0.2.8 │ │    │ │10.0.3.2 │ │         │
│  │ └─────────┘ │    │ └─────────┘ │    │ └─────────┘ │         │
│  │  CNI Plugin │    │  CNI Plugin │    │  CNI Plugin │         │
│  │  kube-proxy │    │  kube-proxy │    │  kube-proxy │         │
│  └─────────────┘    └─────────────┘    └─────────────┘         │
│         ▲                  ▲                  ▲                │
│         └──────────────────┴──────────────────┘                │
│                    Control Plane                               │
│              (API Server, etcd, Scheduler)                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Container Network Interface (CNI)

### What is CNI?

The **Container Network Interface (CNI)** is a framework for dynamically configuring networking resources in containerized environments. It's a Cloud Native Computing Foundation (CNCF) project that standardizes how network interfaces are created and configured for containers.

### How CNI Works

```
┌─────────────────────────────────────────────────────────────┐
│                    CNI Workflow                             │
│                                                             │
│  ┌──────────┐      ADD/DEL      ┌─────────────────────┐    │
│  │ Kubelet  │ ─────────────────►│   CNI Plugin        │    │
│  │          │                   │  (Calico/Cilium/    │    │
│  │ Creates  │                   │   Flannel/etc)      │    │
│  │   Pod    │                   └─────────────────────┘    │
│  └──────────┘                            │                  │
│                                          ▼                  │
│                              ┌─────────────────────┐       │
│                              │  IPAM Plugin        │       │
│                              │  (IP Allocation)    │       │
│                              └─────────────────────┘       │
│                                          │                  │
│                                          ▼                  │
│                              ┌─────────────────────┐       │
│                              │  Network Interface  │       │
│                              │  + Routes + Rules   │       │
│                              └─────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

### CNI Network Models

| Model | Description | Use Case |
|-------|-------------|----------|
| **Overlay (Encapsulated)** | Uses VXLAN/IPsec to encapsulate L2 over L3 | Multi-datacenter, cloud environments |
| **Underlay (Unencapsulated)** | Direct L3 routing with BGP | On-premises, bare metal, high performance |

### Popular CNI Plugins

| Plugin | Type | Best For |
|--------|------|----------|
| **Calico** | BGP-based L3 | Production, network policies, bare metal |
| **Cilium** | eBPF-based | High performance, observability, L7 policies |
| **Flannel** | VXLAN overlay | Simple setups, development clusters |
| **Weave** | Mesh overlay | Easy setup, encryption, smaller clusters |

---

## 3. Kubernetes Service Types

Services provide stable network endpoints for ephemeral pods. Kubernetes offers four service types:

### Service Type Hierarchy

```
                    ┌─────────────────┐
                    │    Service      │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  ClusterIP    │   │   NodePort    │   │ LoadBalancer  │
│  (Internal)   │◄──│  (External)   │◄──│   (Cloud LB)  │
└───────────────┘   └───────────────┘   └───────────────┘
        │                    │                    │
        │                    │                    │
        ▼                    ▼                    ▼
   Internal Only      NodeIP:Port         External IP
   Virtual IP         30000-32767         Cloud Provider
```

### 3.1 ClusterIP (Default)

Exposes the service on a cluster-internal IP. Only accessible from within the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

**Characteristics:**
- Virtual IP from service-cluster-ip-range
- Only accessible within cluster
- Used for internal microservice communication
- Default service type

### 3.2 NodePort

Exposes the service on each Node's IP at a static port (default range: 30000-32767).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080  # Optional: auto-assigned if not specified
```

**Traffic Flow:**
```
External Client ──► NodeIP:30080 ──► kube-proxy ──► Pod:8080
```

**Limitations:**
- Port range limited (30000-32767)
- One service per port across entire cluster
- Client must know node IP
- Node failure affects accessibility

### 3.3 LoadBalancer

Provisions an external load balancer (cloud provider) or MetalLB (bare metal).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: public-web
  annotations:
    metallb.universe.tf/address-pool: web-pool
spec:
  type: LoadBalancer
  selector:
    app: public-web
  ports:
    - port: 80
      targetPort: 8080
  externalTrafficPolicy: Local  # Preserves client source IP
```

**Traffic Flow:**
```
Client ──► Cloud LB/MetalLB ──► Node:NodePort ──► Pod
```

**External Traffic Policy Options:**

| Policy | Behavior | Pros | Cons |
|--------|----------|------|------|
| `Cluster` (default) | Can forward to pods on any node | Even load distribution | Extra hop, source IP masked |
| `Local` | Stays on receiving node | Preserves source IP, no extra hop | Uneven distribution |

### 3.4 ExternalName

Maps service to external DNS name (CNAME record).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: my.database.example.com
```

### Service Comparison

| Feature | ClusterIP | NodePort | LoadBalancer | ExternalName |
|---------|-----------|----------|--------------|--------------|
| Accessibility | Internal | External via node | External via LB | External DNS |
| Stable IP | Yes | No | Yes | N/A |
| Port Range | Any | 30000-32767 | Any | N/A |
| Cost | Free | Free | Cloud LB cost | Free |
| Use Case | Internal services | Dev/test | Production | External services |

---

## 4. Ingress Controllers

Ingress provides HTTP/HTTPS routing to services based on host and path rules.

### Ingress Architecture

```
                         ┌─────────────────┐
                         │   Internet      │
                         └────────┬────────┘
                                  │
                         ┌────────▼────────┐
                         │  Ingress        │
                         │  Controller     │
                         │  (NGINX/Traefik)│
                         └────────┬────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
              ▼                   ▼                   ▼
       ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
       │  Service A  │    │  Service B  │    │  Service C  │
       │  /api/*     │    │  /web/*     │    │  /*         │
       └─────────────┘    └─────────────┘    └─────────────┘
```

### Example Ingress Resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dashboard-ingress
  namespace: kubernetes-dashboard
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: dashboard.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kubernetes-dashboard
            port:
              number: 80
```

### Popular Ingress Controllers

| Controller | Features | Best For |
|------------|----------|----------|
| **NGINX** | Most popular, mature, extensive features | General purpose |
| **Traefik** | Cloud-native, dynamic config, dashboard | Modern microservices |
| **HAProxy** | High performance, enterprise features | High throughput |
| **Istio Gateway** | Service mesh integration | Advanced traffic management |

---

## 5. Network Policies

NetworkPolicies control traffic flow at OSI Layer 3/4 (IP/port level).

### Default Behavior

By default, **all pods can communicate with all other pods**. NetworkPolicies implement a **default-deny** model where you explicitly allow traffic.

### Network Policy Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Network Policy Flow                      │
│                                                             │
│   ┌─────────────┐         ┌─────────────┐                  │
│   │   Pod A     │◄───────►│   Pod B     │                  │
│   │  (frontend) │  ALLOW  │  (backend)  │                  │
│   └─────────────┘         └─────────────┘                  │
│          │                         ▲                       │
│          │    ┌─────────────┐      │                       │
│          └───►│  Network    │──────┘                       │
│               │  Policy     │                              │
│               │  (ALLOW from│                              │
│               │   frontend) │                              │
│               └─────────────┘                              │
│                          ▲                                 │
│                          │                                 │
│               ┌─────────────┐                              │
│               │  CNI Plugin │                              │
│               │ (enforces)  │                              │
│               └─────────────┘                              │
└─────────────────────────────────────────────────────────────┘
```

### Example Network Policies

**Default Deny All Ingress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

**Allow Specific Traffic:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

**Allow All Egress:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-egress
spec:
  podSelector: {}
  egress:
  - {}
  policyTypes:
  - Egress
```

### Network Policy Limitations

| Limitation | Workaround |
|------------|------------|
| Cannot target Services by name | Target pods by labels |
| No TLS/HTTPS inspection | Use Service Mesh or Ingress |
| No logging of blocked connections | Use CNI-specific features |
| Cannot block localhost traffic | Use host-level firewall |
| No explicit deny rules | Structure policies as allow-only |

---

## 6. DNS and Service Discovery

### CoreDNS Architecture

CoreDNS is the default cluster DNS server, providing name resolution for services.

```
┌─────────────────────────────────────────────────────────────┐
│                    DNS Resolution Flow                      │
│                                                             │
│  ┌─────────┐    DNS Query     ┌─────────────────────────┐  │
│  │   Pod   │ ────────────────►│      CoreDNS            │  │
│  │         │  my-svc.default  │    (10.96.0.10:53)      │  │
│  │         │◄─────────────────│                         │  │
│  └─────────┘   A: 10.96.5.20  └─────────────────────────┘  │
│        ▲                              │                     │
│        │                              │                     │
│        │    /etc/resolv.conf          │ Watches             │
│        │    nameserver 10.96.0.10     ▼                     │
│        │                       ┌─────────────────────────┐  │
│        │                       │   Kubernetes API        │  │
│        └───────────────────────│   (Services/Endpoints)  │  │
│                                └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### DNS Record Formats

| Record Type | Format | Example |
|-------------|--------|---------|
| **A Record** | `<service>.<namespace>.svc.<cluster-domain>` | `user-api.production.svc.cluster.local` |
| **SRV Record** | `_<port>._<protocol>.<service>` | `_http._tcp.user-api.production.svc.cluster.local` |
| **Pod Record** | `<pod-ip>.<namespace>.pod.<cluster-domain>` | `10-0-1-5.default.pod.cluster.local` |

### CoreDNS Configuration

```yaml
# Corefile - CoreDNS configuration
.:53 {
    errors
    health
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}
```

### DNS Debugging Commands

```bash
# Check DNS resolution from a pod
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default

# View CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Check pod's DNS configuration
kubectl exec <pod> -- cat /etc/resolv.conf

# Test service DNS
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup <service-name>.<namespace>
```

---

## 7. kube-proxy and Load Balancing

### kube-proxy Modes

kube-proxy implements Kubernetes services on each node using different backends:

| Mode | Mechanism | Best For |
|------|-----------|----------|
| **iptables** (default) | iptables rules | Small-medium clusters (<1000 services) |
| **IPVS** | Kernel load balancer | Large clusters (>1000 services) |
| **userspace** | User-space proxy | Legacy, not recommended |

### iptables Mode

```
┌─────────────────────────────────────────────────────────────┐
│                  iptables Mode Flow                         │
│                                                             │
│  Client ──► PREROUTING ──► KUBE-SERVICES chain             │
│                                    │                        │
│                                    ▼                        │
│                            ┌───────────────┐               │
│                            │ DNAT to Pod IP│               │
│                            │ (random)      │               │
│                            └───────────────┘               │
│                                    │                        │
│                                    ▼                        │
│                              Backend Pod                    │
│                                                             │
│  Limitation: O(n) rule matching, degrades with scale       │
└─────────────────────────────────────────────────────────────┘
```

### IPVS Mode

IPVS (IP Virtual Server) provides kernel-level load balancing with O(1) lookup.

```
┌─────────────────────────────────────────────────────────────┐
│                    IPVS Mode Flow                           │
│                                                             │
│  Client ──► IPVS Virtual Server ──► Backend Pods           │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              IPVS Scheduling Algorithms              │   │
│  ├─────────────────────────────────────────────────────┤   │
│  │  rr    - Round Robin (default)                      │   │
│  │  lc    - Least Connection                           │   │
│  │  sh    - Source Hashing (session affinity)          │   │
│  │  wrr   - Weighted Round Robin                       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  Benefits: O(1) lookup, better performance at scale        │
└─────────────────────────────────────────────────────────────┘
```

### Enabling IPVS Mode

```yaml
# kube-proxy ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-proxy
  namespace: kube-system
data:
  config.conf: |
    apiVersion: kubeproxy.config.k8s.io/v1alpha1
    kind: KubeProxyConfiguration
    mode: "ipvs"
    ipvs:
      scheduler: "rr"
      strictARP: true  # Required for MetalLB
```

### IPVS Prerequisites

```bash
# Load kernel modules
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack

# Install ipvsadm for debugging
apt-get install ipvsadm  # Debian/Ubuntu
yum install ipvsadm      # RHEL/CentOS

# Verify IPVS is active
ipvsadm -L -n
```

---

## 8. Popular CNI Plugins Comparison

### Feature Comparison Matrix

| Feature | Calico | Cilium | Flannel | Weave |
|---------|--------|--------|---------|-------|
| **Network Policy** | Full + Extended | Full + L7 | None | Full |
| **Data Plane** | iptables/eBPF | eBPF | VXLAN | Mesh Overlay |
| **Encryption** | WireGuard | WireGuard/IPsec | No | NaCl |
| **Observability** | Basic | Hubble (Advanced) | None | Basic |
| **Performance** | High | Very High | Medium | Medium |
| **Complexity** | Medium | Medium-High | Low | Low-Medium |
| **L7 Policies** | No (OSS) | Yes | No | No |
| **BGP Support** | Yes | Yes | No | No |

### 8.1 Calico

**Best for:** Production environments, network policies, bare metal deployments

```bash
# Install Calico
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
```

**Key Features:**
- BGP-based routing (no overlay in L3 mode)
- Full Network Policy support + GlobalNetworkPolicy
- WireGuard encryption
- Integration with physical network infrastructure
- Battle-tested, widely deployed

**Deployment Modes:**
| Mode | Description |
|------|-------------|
| **Direct Routing** | BGP routes, no encapsulation, best performance |
| **VXLAN Overlay** | Encapsulated, works anywhere |
| **IP-in-IP** | Legacy overlay mode |

### 8.2 Cilium

**Best for:** High performance, observability, advanced security, eBPF-compatible kernels (5.10+)

```bash
# Install Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz
tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin

# Install Cilium
cilium install --version 1.15.0
cilium status --wait
```

**Key Features:**
- eBPF-based data plane (bypasses iptables)
- Layer 7 policies (HTTP, gRPC, Kafka)
- Hubble observability
- Kube-proxy replacement
- Service mesh capabilities (without sidecars)
- Cluster Mesh for multi-cluster

**Cilium Architecture:**
```
┌─────────────────────────────────────────────────────────────┐
│                    Cilium Components                        │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ Cilium Agent│  │   Cilium    │  │    CNI      │         │
│  │ (per node)  │  │  Operator   │  │   Plugin    │         │
│  └──────┬──────┘  └─────────────┘  └─────────────┘         │
│         │                                                   │
│         ▼                                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    eBPF Programs                     │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌────────┐ │   │
│  │  │   XDP   │  │   TC    │  │ Socket  │  │Conntrack│ │   │
│  │  │(NIC Lvl)│  │(Netdev) │  │ (L7)    │  │        │ │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └────────┘ │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Hubble Observability                    │   │
│  │         (Flow logs, metrics, UI)                     │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 8.3 Flannel

**Best for:** Simple setups, development clusters, minimal operational overhead

```bash
# Install Flannel
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

**Key Features:**
- Simple VXLAN overlay
- Easy setup and maintenance
- No network policies
- Good for small clusters

### 8.4 Weave

**Best for:** Easy setup with encryption, smaller clusters

```bash
# Install Weave
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

**Key Features:**
- Automatic mesh topology
- Built-in encryption (NaCl)
- Handles network partitions gracefully
- Full Network Policy support

---

## 9. Best Practices

### General Networking Best Practices

1. **Choose the Right CNI for Your Use Case**
   - Small/dev clusters: Flannel or Weave
   - Production with policies: Calico
   - High performance/observability: Cilium

2. **Use IPVS for Large Clusters**
   - Switch from iptables to IPVS when >1000 services
   - Better performance and scalability

3. **Implement Network Policies**
   - Start with default-deny
   - Explicitly allow required traffic
   - Regular policy audits

4. **DNS Optimization**
   - Scale CoreDNS for large clusters
   - Use dnsPolicy: None for custom configurations
   - Monitor DNS query latency

5. **Service Design**
   - Use ClusterIP for internal communication
   - Use Ingress for HTTP/HTTPS traffic
   - Use LoadBalancer for TCP/UDP services

### Security Best Practices

```yaml
# Default deny all ingress in namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
# Allow specific traffic only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### Performance Best Practices

| Area | Recommendation |
|------|----------------|
| **CNI** | Use native routing mode when possible |
| **kube-proxy** | Enable IPVS for large clusters |
| **DNS** | Scale CoreDNS replicas based on cluster size |
| **MTU** | Configure proper MTU for overlay networks |
| **Monitoring** | Use Hubble (Cilium) or similar for visibility |

### Troubleshooting Commands

```bash
# Check CNI configuration
ls /etc/cni/net.d/
cat /etc/cni/net.d/*.conf

# Verify pod networking
kubectl exec <pod> -- ip addr
kubectl exec <pod> -- ip route

# Check service endpoints
kubectl get endpoints <service>

# Debug DNS
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default

# Check kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy

# View IPVS rules (if using IPVS mode)
ipvsadm -L -n

# Check network policies
kubectl get networkpolicies --all-namespaces
```

---

## References

- [Kubernetes Official Documentation - Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [CNI Specification](https://github.com/containernetworking/cni)
- [Calico Documentation](https://docs.projectcalico.org/)
- [Cilium Documentation](https://docs.cilium.io/)
- [CoreDNS](https://coredns.io/)

---

*Research compiled on April 2, 2026*
