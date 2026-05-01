*[Leia em Português](README.pt-BR.md)*

# CheckMK Community — Kubernetes Deployment (Homelab)

Deploy of **CheckMK Community Edition 2.5.0** on Kubernetes with NFS-backed persistent storage.

The original YAML was created by my colleague **Norman Sa Silva** for a corporate RKE1 cluster and adapted by me for my homelab (k8s v1.35 / Debian 13 / Calico).

---

## Why a manual PersistentVolume instead of the homelab StorageClass?

The homelab uses the `nfs-homelab` StorageClass (provisioner `nfs.csi.k8s.io`) with **NFSv4.1**, which is the recommended default for modern Kubernetes. However, **CheckMK/OMD has a known file locking issue over NFSv4**: the internal processes use `flock()` extensively for inter-service locking (Nagios, Apache, rrdcached, etc.), and the Linux kernel, when operating over NFSv4 without `local_lock`, forwards those calls to the NFS server — resulting in `OSError: [Errno 9] Bad file descriptor` and a container crash loop.

The official CheckMK documentation acknowledges this behavior for containerized environments with network volumes:

> *"If you use an NFS volume, make sure that file locking works correctly. Some NFS configurations do not support POSIX file locking, which can cause issues with the OMD site."*
> — [CheckMK Docker documentation](https://docs.checkmk.com/latest/en/introduction_docker.html)

### Solution: manual PV with NFSv3 + nolock + local_lock=all

Using a manual `PersistentVolume` allows mount options to be specified directly on the PV object — unlike dynamic CSI provisioning, which applies options at the StorageClass level (affecting all workloads using it). With a dedicated PV for CheckMK, the options are fully isolated:

```yaml
mountOptions:
  - vers=3          # NFSv3: no NFSv4 locking issue with flock()
  - nolock          # Do not send lock requests to the NFS server
  - local_lock=all  # Resolve all locking (flock + posix) locally on the node
  - hard
  - noatime
  - rsize=1048576
  - wsize=1048576
```

**Why not modify the existing StorageClass?**
The `nfs-homelab` StorageClass is shared by other workloads (n8n, postgres, cotacoes, etc.) that work correctly with NFSv4.1. Changing the mount options globally could break those services — and `local_lock=all` is not recommended for multi-pod workloads that rely on real distributed locking. The manual PV keeps CheckMK isolated with exactly the configuration it needs.

**Why does NFSv3 work?**
With NFSv3, using `flock()` with `nolock` makes the kernel resolve locking entirely on the client side (the K8s node), without contacting the NFS server. Since CheckMK runs as a single pod (`strategy: Recreate`), there is no risk of conflict between instances — local locking is both sufficient and correct.

---

## Deploy structure

```
checkmk.deploy.yaml
├── Namespace
├── PersistentVolume       # Manual PV — NFSv3 with dedicated mount options
├── PersistentVolumeClaim  # Direct bind to the PV (storageClassName: "")
├── ConfigMap              # siteid + timezone
├── Secret                 # CMK_PASSWORD
├── Deployment             # 1 replica, Recreate strategy
│   ├── emptyDir (tmpfs)   # /opt/omd/sites/cmk/tmp — volatile, in-memory
│   └── NFS PVC            # /omd/sites — persistent
├── Service (ClusterIP)    # Port 5000 — internal access
├── Service (LoadBalancer) # Port 8000 — Agent Receiver (MetalLB, TCP)
└── Ingress                # nginx + cert-manager (Let's Encrypt)
```

### Why emptyDir for /tmp?

The `/opt/omd/sites/cmk/tmp` directory holds volatile state files (sockets, PIDs, checkresults, status.dat). The official documentation recommends mounting a tmpfs there for containerized deployments:

> *"Mount a tmpfs to /opt/omd/sites/\<site\>/tmp to improve performance and avoid unnecessary writes to the persistent volume."*
> — [CheckMK Docker documentation](https://docs.checkmk.com/latest/en/introduction_docker.html)

Without the emptyDir, those files would land on NFS, increasing I/O latency and generating unnecessary writes on every check cycle.

---

## Prerequisites

- Kubernetes 1.28+
- NGINX Ingress Controller
- cert-manager with ClusterIssuer `prod-letsencrypt-cloudflare`
- Accessible NFS server (update `server` and `path` in the PV spec)
- NFS directory created on the server beforehand:
  ```bash
  mkdir -p /mnt/nfs-data/k8s/checkmk
  chmod 777 /mnt/nfs-data/k8s/checkmk
  ```

---

## Applying the manifest

```bash
# Edit the Ingress host and the NFS server/path in the PV before applying
kubectl apply -f checkmk.deploy.yaml

# Follow the first startup (omd site creation can take 2-3 minutes)
kubectl logs -n checkmk -f deployment/checkmk

# Wait for the pod to become Ready (startupProbe tolerates up to 10 minutes)
kubectl get pod -n checkmk -w
```

Access: `https://checkmk-k8s.staypuff.info/cmk/` — user `cmkadmin`, password as defined in the Secret.

---

## Key environment variables

| Variable | Value | Notes |
|---|---|---|
| `CMK_SITE_ID` | `cmk` | OMD site name |
| `CMK_PASSWORD` | base64 | Password for `cmkadmin` |
| `TZ` | `Brazil/East` | **Do not use** `America/Sao_Paulo` — the entrypoint.sh uses `_` as a separator in a `sed` command, which breaks the site restart |

---

## Troubleshooting: legacy agents not collecting (pull mode)

This section documents a real issue encountered during deployment and the full investigation that led to the fix.

### Symptom

The pod started normally and the CheckMK UI was accessible, but monitored hosts appeared as `DOWN` with no data. The diagnostic command returned:

```
ERROR [agent]: Communication failed: timed out
```

### Context: how pull mode works

CheckMK supports two agent communication modes:

| Mode | Direction | Port |
|---|---|---|
| **Pull (legacy)** | CheckMK server → monitored host | 6556 (on the host) |
| **Push (Agent Controller)** | Monitored host → CheckMK server | 8000 (on the server) |

In pull mode, **the server initiates the connection**. The agent listens on port 6556 of the monitored host and responds with data. No port needs to be exposed on the pod for pull mode.

---

### Step-by-step investigation

#### Step 1 — Verify the pod can ping hosts

```bash
sudo kubectl exec -it -n checkmk <pod> -- bash
ping -c2 192.168.3.31
```

**Result:** ping worked. Basic networking was fine, but CheckMK needed `NET_RAW` capability to send ICMP — added to the container:

```yaml
securityContext:
  capabilities:
    add: ["NET_RAW"]
```

#### Step 2 — Verify TCP connectivity on the agent port

```bash
# From inside the pod
nc -zw3 192.168.3.11 6556; echo EXIT:$?
```

**Result:** `EXIT:0` — the TCP connection was established. The problem was not a firewall or routing issue.

#### Step 3 — Verify if the agent sends data

```bash
# From inside the pod
timeout 5 bash -c 'cat < /dev/tcp/192.168.3.11/6556' | head -3
```

**Result:** no data received. TCP connects but the agent does not respond.

#### Step 4 — Inspect the agent state on the monitored host

```bash
# On PVE1 (192.168.3.11)
cmk-agent-ctl status --json
```

**Result:**
```json
{
  "version": "2.4.0p25",
  "agent_socket_operational": true,
  "ip_allowlist": [],
  "allow_legacy_pull": true,
  "connections": []
}
```

The agent had `allow_legacy_pull: true` and `ip_allowlist: []` (no IP restriction). `only_from` was not the blocker.

#### Step 5 — Confirm the agent works from other hosts

```bash
# From k8s-master (192.168.3.30)
timeout 5 bash -c 'cat < /dev/tcp/192.168.3.11/6556' | head -3
```

**Result:**
```
<<<check_mk>>>
Version: 2.4.0p25
AgentOS: linux
```

It worked from the master but not from the pod. Same port, same host — different behaviour.

#### Step 6 — Packet capture (tcpdump) — the definitive proof

A `tcpdump` was running on PVE1 while the pod attempted to connect:

```bash
# On PVE1
tcpdump -n -i nic0 port 6556

# Simultaneously, in the pod
timeout 5 bash -c 'cat < /dev/tcp/192.168.3.11/6556'
```

**Captured evidence:**

```
# Connection from the OLD SERVER (192.168.3.5) — works normally:
09:02:24 IP 192.168.3.5.42496  > 192.168.3.31.6556:  Flags [S]   # SYN
09:02:24 IP 192.168.3.31.6556  > 192.168.3.5.42496:  Flags [S.]  # SYN-ACK
09:02:24 IP 192.168.3.5.42496  > 192.168.3.31.6556:  Flags [.]   # ACK
09:02:26 IP 192.168.3.31.6556  > 192.168.3.5.42496:  Flags [.]   # data flowing (183KB)

# Connection from the POD (192.168.194.91) — PROBLEM:
09:02:25 IP 192.168.3.11.6556  > 192.168.194.91.37990: Flags [S.] # SYN-ACK (1st attempt)
09:02:26 IP 192.168.3.11.6556  > 192.168.194.91.37990: Flags [S.] # SYN-ACK (retransmit)
09:02:26 IP 192.168.3.11.6556  > 192.168.194.91.37990: Flags [S.] # SYN-ACK (retransmit)
09:02:27 IP 192.168.3.11.6556  > 192.168.194.91.37990: Flags [S.] # SYN-ACK (retransmit)
09:02:28 IP 192.168.3.11.6556  > 192.168.194.91.37990: Flags [S.] # SYN-ACK (retransmit)
09:02:30 IP 192.168.3.11.6556  > 192.168.194.91.37990: Flags [S.] # SYN-ACK (retransmit)
```

**Diagnosis:** PVE1 never received the ACK from the pod. The TCP handshake never completed — SYN-ACK was sent but the pod never received it (infinite retransmissions).

---

### Root cause: CIDR overlap + Calico without MASQUERADE

The cluster uses Calico with pod CIDR `192.168.0.0/16`.

The homelab physical network uses `192.168.3.0/24` — which falls **inside** the `192.168.0.0/16` range.

When the pod (IP `192.168.194.91`) sent a packet to PVE1 (`192.168.3.11`):

```
Pod 192.168.194.91 ──SYN──▶ PVE1 192.168.3.11
```

Calico interpreted the destination `192.168.3.x` as **belonging to the pod CIDR** and **did not apply MASQUERADE (SNAT)**. The packet arrived at PVE1 with the pod's real source IP (`192.168.194.91`).

PVE1 tried to reply:

```
PVE1 192.168.3.11 ──SYN-ACK──▶ 192.168.194.91 ???
```

But PVE1 had no route to `192.168.194.0/24` — that subnet only exists inside the Kubernetes cluster. The SYN-ACK was sent to the default gateway and dropped. The handshake never completed.

This explained why `nc -z` sometimes returned 0 (the SYN reached PVE1), but no data was ever received (the handshake did not complete).

> **Why did it work from the master/worker nodes?**
> Connections from `192.168.3.30` or `192.168.3.31` use physical IPs. PVE1 knows how to route the response back to `192.168.3.x` over the local network — the handshake completes normally.

---

### Fix: `hostNetwork: true`

With `hostNetwork: true`, the pod has no pod IP (`192.168.194.x`). It uses the IP of the node it is scheduled on (`192.168.3.31`) directly.

```
Pod using 192.168.3.31 ──SYN──▶ PVE1 192.168.3.11
PVE1 192.168.3.11 ──SYN-ACK──▶ 192.168.3.31  ✓ (known route)
```

The `dnsPolicy: ClusterFirstWithHostNet` field is required so the pod continues resolving internal cluster service names (e.g. `checkmk.svc.cluster.local`) even while using the host network.

```yaml
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
```

#### Verification after the fix

```bash
# Pod IP is now the node IP
sudo kubectl get pods -n checkmk -o wide
# NAME                       READY   STATUS    IP             NODE
# checkmk-547c56b7bd-m27xd   1/1     Running   192.168.3.31   k8s-worker1

# Collection working
su - cmk -c "cmk -vd PVE1"
# <<<check_mk>>>
# Version: 2.4.0p25
# AgentOS: linux
# Hostname: pve1
# ...
```

#### Alternatives considered and discarded

| Alternative | Reason discarded |
|---|---|
| Static routes on each monitored host | Fragile: the pod can move to a different node between restarts |
| Changing the cluster pod CIDR | Requires a full cluster rebuild |
| BGP peering between Calico and the physical router | Unnecessary complexity for a homelab |

---

## References

- [CheckMK Docker — official documentation](https://docs.checkmk.com/latest/en/introduction_docker.html)
- [CheckMK Docker Hub](https://hub.docker.com/r/checkmk/check-mk-community)
- [OMD — Open Monitoring Distribution](https://omdistro.org/)
- [Calico — pod CIDR and MASQUERADE](https://docs.tigera.io/calico/latest/networking/configuring/bgp)
- [Kubernetes hostNetwork](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#host-namespaces)
