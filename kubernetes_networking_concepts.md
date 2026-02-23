# Kubernetes Networking Concepts

---

## Overview

This document provides an in-depth exploration of Kubernetes networking — how pods, services, and external traffic communicate within a cluster, the models and abstractions involved, and the components that make it work.

---

## 1. The Kubernetes Networking Model

Kubernetes imposes a simple but powerful networking model defined by these **four fundamental requirements**:

| Requirement | Description |
| ----------- | ----------- |
| **1. All pods can communicate with all other pods without NAT** | Every pod gets its own IP address. Pod A can reach Pod B directly using that IP, regardless of which node they run on. No network address translation between pods. |
| **2. All nodes can communicate with all pods without NAT** | A process on a node can reach any pod using the pod's IP. Useful for debugging, monitoring, and host-to-pod communication. |
| **3. A pod sees its own IP the same way others see it** | No "from inside vs outside" distinction for the pod. The IP a pod uses for itself is the same IP others use to reach it. |
| **4. Cluster IP is virtual** | Services get a stable virtual IP. kube-proxy (or equivalent) forwards traffic from that IP to backend pods. The Service IP does not exist on any interface. |

```text
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         KUBERNETES NETWORKING MODEL                              │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   Node A                    Node B                    Node C                    │
│   ┌──────────────┐          ┌──────────────┐          ┌──────────────┐         │
│   │ Pod 10.0.1.1 │◄────────►│ Pod 10.0.2.1 │◄────────►│ Pod 10.0.3.1 │         │
│   │ Pod 10.0.1.2 │   no     │ Pod 10.0.2.2 │   no     │ Pod 10.0.3.2 │         │
│   └──────────────┘   NAT    └──────────────┘   NAT    └──────────────┘         │
│         │                         │                         │                   │
│         └─────────────────────────┼─────────────────────────┘                   │
│                                   │                                             │
│                     All pods reachable by pod IP across nodes                    │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Pod Networking

### 2.1 Pod IP Allocation

Each pod gets a **unique IP address** from a CIDR range assigned to the node. This is typically delegated by the **CNI (Container Network Interface)** plugin.

```text
┌─────────────────────────────────────────────────────────────────┐
│  POD NETWORK LAYOUT (typical setup)                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Cluster CIDR:     10.244.0.0/16  (or 10.96.0.0/12 for services)│
│                                                                 │
│  Node subnet:      10.244.1.0/24  (Node 1)                      │
│                    10.244.2.0/24  (Node 2)                      │
│                    10.244.3.0/24  (Node 3)                      │
│                                                                 │
│  Pod A on Node 1:  10.244.1.5                                   │
│  Pod B on Node 1:  10.244.1.6                                   │
│  Pod C on Node 2:  10.244.2.3                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Pod Network Namespace

Each pod has its own **network namespace**. Inside it:

- A virtual Ethernet pair (`veth`) connects the pod’s namespace to the node’s (or a bridge).
- The pod sees its own IP on one end of the pair; the other end is in the host/bridge namespace.
- The CNI plugin sets up routing, bridges, or overlay tunnels so traffic can reach other pods and nodes.

```text
┌─────────────────────────────────────────────────────────────────┐
│  POD NETWORK NAMESPACE                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   HOST NAMESPACE                    POD NAMESPACE                │
│   ┌─────────────────────┐           ┌─────────────────────┐     │
│   │                     │   veth    │                     │     │
│   │   cbr0 (bridge)     │◄─────────►│  eth0  (10.244.1.5) │     │
│   │   or veth-host      │           │                     │     │
│   │                     │           │  container process   │     │
│   └─────────────────────┘           └─────────────────────┘     │
│            │                                                        │
│            │  routing / overlay                                    │
│            ▼                                                        │
│   Other nodes / pods                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 Container-to-Container (same pod)

Containers in the same pod share the **same network namespace**. They all use the same IP and can talk to each other via `localhost`.

```text
┌─────────────────────────────────────────┐
│  Pod (single network namespace)         │
│  ┌─────────────┐  ┌─────────────┐      │
│  │ Container A │  │ Container B │      │
│  │             │  │             │      │
│  │ localhost   │◄►│ localhost   │      │
│  │ :8080       │  │ :9090       │      │
│  └─────────────┘  └─────────────┘      │
│          \           /                  │
│           \         /                   │
│            └───┬───┘                    │
│                │ eth0 (10.244.1.5)      │
└─────────────────────────────────────────┘
```

---

## 3. Service Networking

Pods are ephemeral; their IPs change when they are recreated. **Services** provide a stable endpoint for accessing a set of pods.

### 3.1 Service Object

A Service is an abstraction that defines a logical set of pods and a policy to access them. It has:

| Field | Purpose |
| ----- | ------- |
| **ClusterIP** | Virtual IP (default). Only reachable inside the cluster. |
| **Selector** | Label selector to find backend pods. |
| **Port(s)** | Port on which the Service listens and target port on pods. |
| **Endpoints** | Dynamically updated list of pod IP:port pairs that match the selector. |

### 3.2 Service Types

```text
┌────────────────────────────────────────────────────────────────────────────────┐
│  SERVICE TYPES                                                                  │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ClusterIP (default)         NodePort                 LoadBalancer             │
│  ┌─────────────────┐        ┌─────────────────┐     ┌─────────────────┐      │
│  │ 10.96.x.x       │        │ Node IP:30080   │     │ External LB     │      │
│  │ (cluster only)  │        │ (NodeIP:Port)   │     │ (cloud provider) │      │
│  └────────┬────────┘        └────────┬────────┘     └────────┬────────┘      │
│           │                          │                       │                │
│           └──────────────────────────┼───────────────────────┘                │
│                                      ▼                                         │
│                            ┌─────────────────┐                                 │
│                            │   Pod Endpoints │                                 │
│                            └─────────────────┘                                 │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

| Type | Use Case | Access |
| ---- | -------- | ------ |
| **ClusterIP** | Internal service-to-service communication | Only from within the cluster |
| **NodePort** | Expose a service on each node’s IP at a fixed port (30000–32767) | NodeIP:NodePort from anywhere |
| **LoadBalancer** | Cloud-integrated external load balancer | External IP/DNS from internet |
| **ExternalName** | Map a Service to an external DNS name (CNAME) | Resolves to external hostname |

### 3.3 ClusterIP — Virtual IP

ClusterIP is a **virtual IP**. It does not exist on any NIC. `kube-proxy` (or a service mesh sidecar) programs **iptables** or **IPVS** rules so that traffic to the ClusterIP is **DNAT**ed to a backend pod IP:port.

```text
┌─────────────────────────────────────────────────────────────────┐
│  CLUSTERIP FLOW                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client pod sends:  GET http://10.96.1.5:80                     │
│                                                                 │
│  ┌─────────────┐    iptables/IPVS    ┌─────────────────────┐    │
│  │ Client Pod  │ ──────────────────► │ Service 10.96.1.5   │    │
│  │ 10.244.1.3  │    DNAT to one of   │ Endpoints:          │    │
│  └─────────────┘    the endpoints   │ - 10.244.1.5:8080   │    │
│                                     │ - 10.244.2.3:8080   │    │
│                                     │ - 10.244.3.1:8080   │    │
│                                     └──────────┬──────────┘    │
│                                                │               │
│                                     Load balancing (random,    │
│                                     round-robin, session affinity)│
│                                                ▼               │
│                                     ┌─────────────────────┐    │
│                                     │ Backend Pod         │    │
│                                     │ 10.244.2.3:8080     │    │
│                                     └─────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.4 Endpoints and EndpointSlices

- **Endpoints**: A resource that lists all pod IP:port pairs matching a Service selector. Updated by the control plane as pods come and go.
- **EndpointSlices**: A scaled replacement for Endpoints. Large Services are split into multiple slices (up to 100 endpoints per slice) for better performance.

---

## 4. DNS in Kubernetes

Kubernetes includes a built-in DNS server (CoreDNS, or previously kube-dns) so workloads can resolve Services and pods by name.

### 4.1 DNS Records Created

| Record Type | Name | Resolves To |
| ----------- | ---- | ----------- |
| **A/AAAA** | `<service>.<namespace>.svc.cluster.local` | ClusterIP of the Service |
| **A/AAAA** | `<service>.<namespace>.svc` | Same (short form) |
| **A/AAAA** | `<service>` (same namespace) | Same |
| **SRV** | `_<port>._<proto>.<service>.<namespace>.svc.cluster.local` | Port and host for headless Services |
| **A/AAAA** (headless) | `<service>.<namespace>.svc.cluster.local` | List of pod IPs |

### 4.2 DNS Resolution Flow

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│  DNS RESOLUTION FLOW                                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Pod /etc/resolv.conf:                                                      │
│  nameserver 10.96.0.10   (kube-dns Service ClusterIP)                       │
│  search default.svc.cluster.local svc.cluster.local cluster.local           │
│                                                                             │
│  App does:  nslookup my-service                                             │
│                                                                             │
│  ┌─────────────┐      DNS query       ┌─────────────┐      lookup          │
│  │ Pod         │ ───────────────────► │ CoreDNS     │ ──────────────────►  │
│  │ (app)       │   my-service         │ (kube-dns)  │   API Server /       │
│  └─────────────┘                      └─────────────┘   in-memory cache    │
│        ▲                                      │                            │
│        │                                      │                            │
│        └──────────── A record: 10.96.1.5 ─────┘                            │
│                                                                             │
│  Short names:  my-service  →  my-service.default.svc.cluster.local          │
│  Cross-namespace:  my-service.other-ns  →  full FQDN                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 Pod DNS (optional)

With `hostname` and `subdomain` set on a pod, and a headless Service with the same name as the subdomain, a pod can get a DNS record:

- `hostname.subdomain.namespace.svc.cluster.local` → pod IP

---

## 5. kube-proxy and Data Plane

`kube-proxy` runs on every node and implements the Service abstraction by programming the local netfilter/iptables or IPVS rules.

### 5.1 Modes

| Mode | Mechanism | Pros | Cons |
| ---- | --------- | ---- | ---- |
| **iptables** | netfilter DNAT rules | Simple, no extra daemon | O(n) rule traversal; slower for many Services |
| **ipvs** | Linux IP Virtual Server (LVS) | O(1) lookup; better for large clusters | Requires ip_vs modules |
| **kernelspace** (Windows) | Windows HNS | Native on Windows | Windows only |

### 5.2 iptables Chain Structure

```text
┌─────────────────────────────────────────────────────────────────┐
│  KUBE-SERVICES (PREROUTING / OUTPUT)                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  -d 10.96.1.5/32 -p tcp -m tcp --dport 80                       │
│    → KUBE-SVC-XXXXX   (Service chain)                           │
│                                                                 │
│  KUBE-SVC-XXXXX (load balancing across endpoints):              │
│    → 33% KUBE-SEP-AAA  (endpoint 1)                             │
│    → 33% KUBE-SEP-BBB  (endpoint 2)                             │
│    → 33% KUBE-SEP-CCC  (endpoint 3)                             │
│                                                                 │
│  KUBE-SEP-AAA:                                                  │
│    -d 10.244.1.5 -p tcp -m tcp --dport 8080 -j DNAT             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

Traffic to the ClusterIP is matched, then load-balanced to endpoint chains that perform DNAT to the pod IP:port.

---

## 6. CNI — Container Network Interface

Kubernetes does **not** implement pod networking itself. It delegates that to a **CNI plugin**.

### 6.1 CNI Responsibilities

1. **Allocate** an IP to the pod.
2. **Add** the container to the network (create veth, attach to bridge/overlay, set routes).
3. **Delete** the container from the network when the pod is removed.

### 6.2 CNI Plugin Types

```text
┌─────────────────────────────────────────────────────────────────────────────────┐
│  CNI PLUGIN ARCHITECTURES                                                       │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  Bridge/VLAN              Overlay (VXLAN, etc.)        Route-based              │
│  ┌─────────────────┐     ┌─────────────────────┐      ┌─────────────────┐     │
│  │ cbr0 per node   │     │ Encapsulate in      │      │ Direct routing  │     │
│  │ L2 on same node │     │ outer UDP/VXLAN     │      │ Node = router   │     │
│  │ Route between   │     │ Works across L3     │      │ No overlay      │     │
│  │ nodes           │     │ networks           │      │ Simple, fast    │     │
│  └─────────────────┘     └─────────────────────┘      └─────────────────┘     │
│                                                                                 │
│  Examples: Calico (route), Flannel (VXLAN), Cilium (eBPF), Weave, etc.           │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

| Plugin | Model | Notes |
| ------ | ----- | ----- |
| **Flannel** | Overlay (VXLAN) or host-gw | Simple; good for getting started |
| **Calico** | BGP routing or overlay | Policy and performance; BGP for large clusters |
| **Cilium** | eBPF-based | Observability, policy, service mesh integration |
| **Weave** | VXLAN overlay | Encryption option; mesh overlay |
| **AWS VPC CNI** | Native VPC IPs | Pods get real VPC IPs; no overlay |

### 6.3 CNI Configuration

The kubelet reads CNI config from `/etc/cni/net.d/` and binaries from `/opt/cni/bin/`. A typical config:

```json
{
  "cniVersion": "0.4.0",
  "name": "mynet",
  "type": "bridge",
  "bridge": "cbr0",
  "ipam": {
    "type": "host-local",
    "subnet": "10.244.0.0/16"
  }
}
```

---

## 7. Network Policies

**NetworkPolicy** defines allow/deny rules for pod-to-pod (and sometimes external) traffic. By default, Kubernetes allows all traffic; a policy enforces isolation.

### 7.1 Policy Spec Structure

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example-policy
spec:
  podSelector:
    matchLabels:
      role: backend      # Policies apply to these pods
  policyTypes:
  - Ingress              # Incoming traffic
  - Egress               # Outgoing traffic
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: TCP
      port: 53
```

### 7.2 Ingress and Egress Rules

| Direction | Purpose |
| --------- | ------- |
| **Ingress** | Who can reach the selected pods, and on which ports |
| **Egress** | Where the selected pods can send traffic, and on which ports |

Selectors can use:

- `podSelector` — pods in the same or specified namespaces
- `namespaceSelector` — all pods in matching namespaces
- `ipBlock` — CIDR ranges (for external IPs)

**Note**: NetworkPolicy is enforced only if the CNI plugin supports it (e.g., Calico, Cilium). Basic Flannel does not support NetworkPolicy unless used with a backend like Calico.

---

## 8. Ingress

**Ingress** exposes HTTP/HTTPS routes from outside the cluster to Services. It is an API resource; an **Ingress controller** implements it (e.g., NGINX, Traefik, AWS ALB).

### 8.1 Ingress Flow

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│  INGRESS FLOW                                                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Internet                                                                   │
│      │                                                                      │
│      ▼                                                                      │
│  ┌─────────────────┐     Ingress rules    ┌─────────────────┐             │
│  │ Load Balancer   │ ──────────────────►  │ Ingress Controller│             │
│  │ (external)      │   host/path routing  │ (e.g. NGINX)     │             │
│  └─────────────────┘                      └────────┬────────┘             │
│                                                     │                        │
│                                                     │ ClusterIP /            │
│                                                     │ direct pod             │
│                                                     ▼                        │
│                                            ┌─────────────────┐              │
│                                            │ Backend Service │              │
│                                            │ / Pods          │              │
│                                            └─────────────────┘              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Ingress vs Service

| Resource | Layer | Use Case |
| -------- | ----- | -------- |
| **Service** | L4 (TCP/UDP) | Raw TCP/UDP, internal load balancing |
| **Ingress** | L7 (HTTP/HTTPS) | Host-based and path-based routing, TLS termination |

---

## 9. End-to-End Packet Flow

### 9.1 Pod-to-Pod (different nodes)

```text
┌───────────────────────────────────────────────────────────────────────────────────┐
│  POD A (Node 1)  →  POD B (Node 2)                                                 │
├───────────────────────────────────────────────────────────────────────────────────┤
│                                                                                   │
│  1. App in Pod A sends packet to 10.244.2.3:8080                                 │
│  2. Packet exits pod netns via veth → bridge/overlay on Node 1                    │
│  3. Node 1 routing: destination 10.244.2.0/24 → Node 2 (via CNI routing table)   │
│  4. Packet goes over network (VXLAN encap, or direct route depending on CNI)     │
│  5. Node 2 receives, decapsulates if needed, routes to 10.244.2.3                 │
│  6. Packet reaches Pod B's veth and container                                    │
│                                                                                   │
└───────────────────────────────────────────────────────────────────────────────────┘
```

### 9.2 Pod-to-Service-to-Pod

```text
┌───────────────────────────────────────────────────────────────────────────────────┐
│  POD A  →  Service (ClusterIP)  →  POD B                                          │
├───────────────────────────────────────────────────────────────────────────────────┤
│                                                                                   │
│  1. App resolves "my-svc" via DNS → 10.96.1.5                                     │
│  2. App sends packet to 10.96.1.5:80                                              │
│  3. Packet hits OUTPUT chain; kube-proxy iptables/IPVS rules match               │
│  4. DNAT: 10.96.1.5:80 → 10.244.2.3:8080 (chosen endpoint)                       │
│  5. Packet is routed to Pod B as in pod-to-pod flow                              │
│  6. Return path: Pod B replies to Pod A's IP; no SNAT on return (except if       │
│     external traffic policy or masquerading applies)                             │
│                                                                                   │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Troubleshooting Checklist

| Issue | What to Check |
| ----- | ------------- |
| Pod cannot reach another pod | CNI plugin running; routes on node; NetworkPolicy blocking |
| Pod cannot resolve DNS | CoreDNS running; `kube-dns` Service; `/etc/resolv.conf` in pod |
| Service not reachable | Endpoints exist (`kubectl get endpoints`); kube-proxy running; correct port |
| NodePort not accessible | Firewall; `externalTrafficPolicy`; correct NodePort range |
| Ingress not working | Ingress controller running; Ingress resource; backend Service healthy |

### Useful Commands

```bash
# Inspect Service and Endpoints
kubectl get svc, endpoints -n <namespace>

# Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Debug DNS from a pod
kubectl run -it --rm debug --image=busybox -- nslookup kubernetes

# Inspect iptables rules (node)
iptables -t nat -L KUBE-SERVICES -n

# Check CNI
ls /etc/cni/net.d/
ls /opt/cni/bin/
```

---

## Summary

| Component | Purpose |
| --------- | ------- |
| **Pod IP** | Unique IP per pod; direct pod-to-pod reachability |
| **Service** | Stable virtual IP and load balancing for a set of pods |
| **kube-proxy** | Implements Service via iptables/IPVS DNAT |
| **CoreDNS** | Cluster DNS for Service and pod name resolution |
| **CNI** | Allocates pod IPs and connects pods to the network |
| **NetworkPolicy** | Ingress/egress rules for pod traffic (CNI-dependent) |
| **Ingress** | L7 routing from external LB to Services |

Kubernetes networking builds on a flat pod network, virtual Service IPs, DNS-based discovery, and pluggable CNI and Ingress implementations to support scalable, policy-aware communication inside and outside the cluster.
