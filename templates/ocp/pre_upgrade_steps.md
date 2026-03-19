# Pre-Upgrade Checklist — huba (QA Hub Cluster)

> **Purpose**: Verify cluster stability **before** starting an OCP upgrade.
> Every item must be confirmed (✅) or a documented exception provided.

---

## 0. Upgrade Path & Compatibility

| # | Status | Check | Command | Expected | Comment |
|---|--------|-------|---------|----------|---------|
| 0.1 | ⬜ | **Upgrade path available in graph** | `oc adm upgrade` | Target version visible in `Recommended updates` or `Available updates` | Without a path in the upgrade graph you cannot start an upgrade — `oc adm upgrade --to=X.Y.Z` will fail with `version not found in channel` |
| 0.2 | ⬜ | Upgrade channel set correctly | `oc get clusterversion -o json \| jq .spec.channel` | `stable-4.X` (not `fast` or `candidate` for QA) | `fast`/`candidate` = pre-production builds; `stable` = recommended for QA before production |
| 0.3 | ⬜ | **Deprecated API check** | `oc adm upgrade --include-not-recommended 2>&1 \| grep -i "deprecated\|removed"` or Prometheus: `apiserver_requested_deprecated_apis` | No removed APIs in use | OCP minor upgrade removes old API versions; workloads/operators using removed APIs fail immediately after upgrade |
| 0.4 | ⬜ | ACM ↔ OCP compatibility | [ACM support matrix](https://access.redhat.com/articles/6955985) | ACM version supports **target** OCP version | ACM has its own compatibility matrix; incompatible version = MCH de-reconciles after upgrade |
| 0.5 | ⬜ | ACS ↔ OCP compatibility | [ACS support matrix](https://access.redhat.com/articles/6939657) | ACS version supports **target** OCP version | ACS Central has strict compatibility; wrong version = Central startup fail |
| 0.6 | ⬜ | ODF ↔ OCP compatibility | [ODF support matrix](https://access.redhat.com/articles/6520351) | ODF version supports **target** OCP version | ODF is the storage layer; incompatible version = Ceph OSD eviction after OCP upgrade |
| 0.7 | ⬜ | `oc adm upgrade --include-not-recommended` output recorded | Save output | Saved for RCA purposes; `--include-not-recommended` shows blockers and warnings | |

https://access.redhat.com/labs/ocpupgradegraph/update_path/?channel=stable-4.17&arch=x86_64&is_show_hot_fix=false&current_ocp_version=4.17.11&target_ocp_version=4.20.15
> **BLOCKER**: If `oc adm upgrade` does not show the target version → do NOT start the upgrade. First resolve channel / graph availability.

---

## 1. Nodes

| # | Status | Check | Command | Expected | Comment |
|---|--------|-------|---------|----------|---------|
| 1.1 | ⬜ | All nodes `Ready` | `oc get nodes` | All `Ready`, none `NotReady` / `SchedulingDisabled` | NotReady node = scheduler can't place upgrade workloads; hard blocker |
| 1.2 | ⬜ | No node is cordoned | `oc get nodes \| grep SchedulingDisabled` | Empty output | OCP upgrade drains nodes sequentially — pre-cordoned node blocks drain completion |
| 1.3 | ⬜ | Node CPU / Memory utilization | `oc adm top nodes` | No node above 80% CPU or Memory | Upgrade pods (image pulls, MCO renders) add significant transient load |
| 1.4 | ⬜ | No DiskPressure on any node | `oc get nodes -o json \| jq '.items[].status.conditions[] \| select(.type=="DiskPressure")'` | `status: "False"` on all | Disk pressure triggers eviction; upgrade needs free space for new OCI layers |
| 1.5 | ⬜ | No MemoryPressure on any node | `oc get nodes -o json \| jq '.items[].status.conditions[] \| select(.type=="MemoryPressure")'` | `status: "False"` on all | Memory pressure triggers pod eviction; upgraded components may not restart |
| 1.6 | ⬜ | No pending CSRs | `oc get csr \| grep Pending` | Empty output | Pending CSRs indicate unjoined or certificate-rotating nodes — will block upgrade |
| 1.7 | ⬜ | All nodes on the same OCP version | `oc get nodes -o wide` | Consistent version across all nodes | Pre-existing version skew > 1 minor version is unsupported and breaks upgrade |
| 1.8 | ⬜ | Worker nodes (`qahuba-w01`, `qahuba-w02`, `qahuba-w03`) present | `oc get nodes -l node-role.kubernetes.io/worker` | 3 nodes `Ready` | Missing worker = reduced workload capacity during rolling upgrade |
| 1.9 | ⬜ | Master/control-plane nodes present (3x) | `oc get nodes -l node-role.kubernetes.io/master` | 3 nodes `Ready` | OCP drains masters one at a time; < 3 masters means immediate etcd quorum loss |
| 1.10 | ⬜ | All MachineConfigPools `Updated=True` | `oc get mcp` | `UPDATED=True` for all MCPs | In-progress MachineConfig apply = node rebooting; upgrade on top causes cascading MCO render conflict |
| 1.11 | ⬜ | No MCP `Updating` or `Degraded` | `oc get mcp \| grep -v "True.*False.*False"` | Empty output | Degraded MCP = nodes failed to apply MachineConfig; upgrade would add a second failure on top |
| 1.12 | ⬜ | No MCP paused | `oc get mcp -o json \| jq '.items[] \| select(.spec.paused==true) \| .metadata.name'` | Empty output | Paused MCP = OCP can't drain or update affected nodes; upgrade gets stuck |
| 1.13 | ⬜ | NTP sync healthy on all nodes | Per node: `chronyc tracking` or `timedatectl` | `Reference ID` = `10.11.66.6`, offset < 1s | Clock skew causes TLS cert validation failures and ETCD election instability during upgrade |
| 1.14 | ⬜ | **No PDB with 0 allowed disruptions** | `oc get pdb -A -o json \| jq '.items[] \| select(.status.disruptionsAllowed==0) \| {name:.metadata.name,ns:.metadata.namespace,allowed:.status.disruptionsAllowed}'` | Empty output | PDB with `ALLOWED DISRUPTIONS=0` blocks `node drain` — worker node upgrade gets stuck indefinitely; **most common blocker of the node upgrade phase** |

---

## 2. Cluster Operators

| # | Status | Check | Command | Expected | Comment |
|---|--------|-------|---------|----------|---------|
| 2.1 | ⬜ | All COs `Available=True` | `oc get co` | `AVAILABLE=True` for every operator | Degraded CO before upgrade = double-fault risk; upgrade may not fix and will make diagnosis harder |
| 2.2 | ⬜ | No CO `Degraded=True` | `oc get co \| grep -v "True.*False.*False"` | Empty output | Red Hat support requires all COs healthy before a supported upgrade; standard pre-upgrade gate |
| 2.3 | ⬜ | No CO `Progressing=True` | `oc get co` | `PROGRESSING=False` for all | In-flight CO change + upgrade = race condition; wait for in-progress rollout to complete first |
| 2.4 | ⬜ | Full CO version table | `oc get co -o wide` | Save as baseline for post-upgrade comparison | Captures pre-upgrade state for diff; confirms every CO upgraded to new version post-upgrade |

> If any CO is degraded, **document the root cause** and confirm it is pre-existing (not introduced by recent changes) before proceeding.

---

## 3. ETCD

| # | Status | Check | Command | Expected | Comment |
|---|--------|-------|---------|----------|---------|
| 3.1 | ⬜ | ETCD cluster health | From etcd pod: `etcdctl endpoint health --cluster -w table` | All 3 endpoints `healthy` | OCP upgrade drains one master at a time; unhealthy member = immediate quorum loss when that master is drained |
| 3.2 | ⬜ | ETCD member list | `etcdctl member list -w table` | 3 members, all `started`, correct IPs | Missing member = degraded quorum; upgrade would bring quorum below threshold |
| 3.3 | ⬜ | ETCD leader elected (one only) | `etcdctl endpoint status --cluster -w table` | Exactly 1 `IS LEADER=true` | No leader = writes blocked cluster-wide; upgrade cannot proceed without write access to API |
| 3.4 | ⬜ | ETCD Raft index lag | Same command above, check `RAFT INDEX` vs `APPLIED INDEX` | Lag ≤ 1000 across all members | High lag = member actively catching up; upgrade during catch-up increases data loss window |
| 3.5 | ⬜ | No ETCD alarms | `etcdctl alarm list` | Empty output | NOSPACE alarm blocks all writes; upgrade is impossible while alarm is active |
| 3.6 | ⬜ | ETCD CO health | `oc get co etcd` | `Available=True, Degraded=False` | High-level OCP health gate on top of etcdctl checks |
| 3.7 | ⬜ | ETCD backup schedule configured | `ls ocp-qahuba01/backup/etcd/` | `huba-values.yaml` exists and matches cluster | Confirms automated backup (runs hourly) exists before a destructive change |
| 3.8 | ⬜ | ETCD OBC `etcd-huba-backup` bound | `oc get obc etcd-huba-backup -n apc-backup` | `Bound` | OBC is the S3 target for ETCD backups; unbound = backups silently failing |
| 3.9 | ⬜ | Last ETCD backup completed | `oc get jobs -n apc-backup --sort-by=.metadata.creationTimestamp \| tail -5` | Most recent job `Completed` | Confirms backup infrastructure working before upgrade; last-known-good before state change |
| 3.10 | ⬜ | LVMCluster healthy on masters | `oc get lvmcluster -n openshift-storage` | `Ready` | ETCD runs on TopoLVM logical volumes (`lvm-etcd-0/1/2`) — LVM failure = ETCD data loss |
| 3.11 | ⬜ | ETCD logical volumes present (10Gi each) | `oc get logicalvolume -l component=lvm-etcd` | 3 LVs `Ready`, each `10Gi` on correct master node | Verifies dedicated ETCD storage volumes haven't been deleted or moved |
| 3.12 | ⬜ | `/var/lib/etcd` mounted on all masters | Per master: `ssh core@<master> systemctl is-active var-lib-etcd.mount` | `active` on all 3 masters | If mount unit failed, ETCD writes to root tmpfs — data would be lost on node reboot |
| 3.13 | ⬜ | MD RAID `/dev/md/mdlvm` healthy on masters | Per master: `ssh core@<master> cat /proc/mdstat` | All devices `UU` (active), no failed members | RAID underpins the LVM VG; degraded RAID = LVM PV at risk = ETCD data loss risk |
| 3.14 | ⬜ | **ETCD DB size < 500 MB** | `etcdctl endpoint status --cluster -w table` → `DB SIZE` column | All members below 500 MB | Large DB slows snapshots and complicates upgrade; if > 500 MB, run defragmentation before upgrade |
| 3.15 | ⬜ | ETCD defragmentation (if DB > 500 MB) | `etcdctl defrag --cluster` | Completed without error; DB size will decrease | Safe operation before upgrade; recommended preventively if DB > 300 MB |

```bash
# Setup env inside etcd pod (exec into etcd container on master node):
export ETCDCTL_ENDPOINTS='https://localhost:2379'
export ETCDCTL_CACERT='/etc/etcd/tls/etcd-ca/ca.crt'
export ETCDCTL_CERT='/etc/etcd/tls/peer/peer.crt'
export ETCDCTL_KEY='/etc/etcd/tls/peer/peer.key'
export ETCDCTL_API=3
```

---

## 4. ArgoCD & GitOps

| # | Status | Check | Command / UI | Expected | Comment |
|---|--------|-------|-------------|----------|---------|
| 4.1 | ⬜ | All ArgoCD apps `Synced` | `oc get apps -n apc-gitops` | `SYNC STATUS=Synced` for all | OutOfSync app = drift between Git and cluster; upgrade may trigger re-sync and revert mid-flight changes |
| 4.2 | ⬜ | All ArgoCD apps `Healthy` | `oc get apps -n apc-gitops` | `HEALTH STATUS=Healthy` for all | Unhealthy app = existing component issue that will be compounded by upgrade restarts |
| 4.3 | ⬜ | No app OutOfSync | `oc get apps -n apc-gitops \| grep -v Synced` | Empty (or only expected exclusions) | Confirms 4.1 more explicitly; expected exclusions must be documented |
| 4.4 | ⬜ | ArgoCD UI accessible | ArgoCD route | Login works, no error banner | UI needed for real-time monitoring during upgrade; broken UI = blind operation |
| 4.5 | ⬜ | `app-of-apps` app health | ArgoCD UI → app-of-apps | `Healthy + Synced` | Parent app drives all child app sync; if unhealthy, child apps won't reconcile after upgrade |
| 4.6 | ⬜ | GitOps repo connected | ArgoCD UI → Settings → Repositories | `conf-sp-qa` repo `Connected` | Disconnected repo = ArgoCD can't pull new manifests; critical for post-upgrade reconciliation |
| 4.7 | ⬜ | `rendered/` is up to date | `make render ENV=huba && git diff rendered/` | No diff (or all changes committed) | Stale rendered manifests = ArgoCD applies old config after upgrade and reverts changes |
| 4.8 | ⬜ | Git working tree clean | `git status` | `working tree clean` after resolving P.1 | Uncommitted file changes = lost if emergency rollback via `git checkout` is needed |

---

## 5. ODF / Ceph (Storage)

| # | Status | Check | Command | Expected | Comment |
|---|--------|-------|---------|----------|---------|
| 5.1 | ⬜ | Ceph overall health | From rook-ceph-tools: `ceph status` | `health: HEALTH_OK` | Storage errors during upgrade = PVC-dependent StatefulSets fail to restart; hard blocker |
| 5.2 | ⬜ | Ceph health detail | `ceph health detail` | No WARN / ERR messages | Even WARN-level issues (e.g., slow OSD) may escalate under upgrade I/O load |
| 5.3 | ⬜ | OSD disk utilization | `ceph osd df` | Every OSD below 75% (`%USE`) | New OCI image layers + ETCD snapshots increase storage during upgrade; headroom required |
| 5.4 | ⬜ | OSD tree — all up/in | `ceph osd tree` | All OSDs `up, in` (6 OSDs expected) | `down` OSD = degraded PG replication; upgrade I/O may trigger data unavailability |
| 5.5 | ⬜ | Ceph full_ratio NOT manually lowered | `ceph osd dump \| grep full_ratio` | `full_ratio 0.85` (default — **not 0.9 or higher!**) | Known issue: `full_ratio` was manually lowered to 0.65 previously → false HEALTH_ERR at 65% |
| 5.6 | ⬜ | No recent Ceph daemon crashes | `ceph crash ls-new` | Empty output | Recent crashes = instability under normal load; upgrade stress may reproduce or worsen the crash |
| 5.7 | ⬜ | NooBaa BackingStore `Ready` | `oc get backingstore -n openshift-storage` | `phase: Ready` | OADP backups use S3 via NooBaa; failing backing store = silent backup failure before upgrade |
| 5.8 | ⬜ | NooBaa / OBC OCSInit healthy | `oc describe ocsinitializations/ocsinit -n openshift-storage` | No error conditions | OCSInit errors indicate partially bootstrapped ODF; underlying issue could worsen post-upgrade |
| 5.9 | ⬜ | All PVCs `Bound` | `oc get pvc -A \| grep -v Bound` | Empty output (no Pending/Lost) | Unbound PVC = pod can't start after rolling restart; upgrade replaces every pod |
| 5.10 | ⬜ | ODF StorageCluster available | `oc get storagecluster -n openshift-storage` | `Phase: Ready` | Top-level ODF health gate; Not Ready = underlying storage subsystem problem |
| 5.11 | ⬜ | ODF Operator CO | `oc get co \| grep odf` | `Available=True` | CO-level health; ODF CO degraded = storage operator itself has issues |

```bash
# Enable CephTools pod if not active:
oc patch -n openshift-storage storagecluster ocs-storagecluster \
  --type json \
  --patch '[{ "op": "replace", "path": "/spec/enableCephTools", "value": true }]'
# Exec into tools:
oc -n openshift-storage rsh $(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name)
```

---

## 6. Vault (HA Raft — 3 replicas)

| # | Status | Check | Command / UI | Expected | Comment |
|---|--------|-------|-------------|----------|---------|
| 6.1 | ⬜ | All 3 Vault pods running | `oc get pods -n apc-vault` | `3/3 Running` (vault-0, vault-1, vault-2) | All app pods fetch secrets from Vault on start; missing pod = reduced HA before upgrade restarts |
| 6.2 | ⬜ | No pod in `Init` / `CrashLoopBackOff` | `oc get pods -n apc-vault` | All `Running` | CrashLoop = Vault repeatedly sealing/unsealing; ESO and Crossplane lose secret access |
| 6.3 | ⬜ | Vault **not sealed** | `oc exec -n apc-vault vault-0 -- vault status` | `Sealed: false` | Sealed Vault = entire secret infrastructure down; all ExternalSecrets fail on pod restart |
| 6.4 | ⬜ | Vault HA Raft — 3 peers, 1 leader | `oc exec -n apc-vault vault-0 -- vault operator raft list-peers` | 3 peers, 1 `leader` | Non-leader pods are read-capable; only 3-peer quorum guarantees write availability during upgrade |
| 6.5 | ⬜ | Transit unseal Vault reachable | `curl -sk https://vault.comm.qa.sp.gr8it.cloud:8200/v1/sys/health \| jq .sealed` | `false` (or HTTP 429 = standby is OK) | Transit Vault is the auto-unseal backend; if unreachable, any Vault pod restart = permanent seal |
| 6.6 | ⬜ | Vault route accessible | `curl -sk https://vault.apps.huba.qa.sp.gr8it.cloud/v1/sys/health \| jq .sealed` | `false` | Verifies ingress path from outside; ingress controller is upgraded early and may break routes temporarily |
| 6.7 | ⬜ | Vault KV `apc-platform` accessible | Vault UI or CLI | Secrets readable | Functional end-to-end test; confirms TLS, auth, and policy are all working |
| 6.8 | ⬜ | ESO ClusterSecretStore ready | `oc get clustersecretstore vault-hub-secret-store` | `READY=True` | ESO uses this store for all ExternalSecrets; not ready = all secrets stop refreshing |
| 6.9 | ⬜ | Vault TLS cert — not expiring soon | `oc get cert -n apc-vault` | Expiry > 30 days | Expired cert during upgrade = TLS handshake failure for every component talking to Vault |
| 6.10 | ⬜ | Vault Prometheus metrics | `oc get servicemonitor -n apc-vault` | ServiceMonitor exists | Ensures Vault metrics visible in Prometheus during upgrade for capacity and error monitoring |

---

## 7. Cert-Manager & Certificates

| # | Status | Check | Command | Expected | Comment |
|---|--------|-------|---------|----------|---------|
| 7.1 | ⬜ | cert-manager pods running | `oc get pods -n cert-manager` | All `Running` | cert-manager must be healthy to issue/renew certs triggered by pod restarts during upgrade |
| 7.2 | ⬜ | ClusterIssuer `vault-hub-issuer` ready | `oc get clusterissuer vault-hub-issuer` | `Ready=True` | Issuer is the bridge between cert-manager and Vault; not ready = all cert requests queue up |
| 7.3 | ⬜ | API server certificate | `oc get cert -n openshift-config` | `Ready=True`, not expiring soon | Expired API server cert = kubectl/oc stops working mid-upgrade; critical blocker |
| 7.4 | ⬜ | No certificate in `False` ready state | `oc get cert -A \| grep -v "True"` | Empty (all certs `Ready`) | Not-ready cert = service currently TLS-broken; upgrade restarts pod and makes it visible |
| 7.5 | ⬜ | Check certs expiring within 30 days | `oc get cert -A -o json \| jq '.items[] \| {name:.metadata.name,ns:.metadata.namespace,expiry:.status.notAfter}'` | No cert expiring in < 30 days | Short-lived cert may expire during multi-hour upgrade window; check and renew proactively |
| 7.6 | ⬜ | trust-manager Bundle healthy | `oc get bundle -n apc-trust-manager` | `Ready=True` | trust-manager injects CA bundles into namespaces; broken = restarted pods missing CA trust |
| 7.7 | ⬜ | cert-utils-operator running | `oc get pods -n cert-utils-operator` | `Running` | Watches certs and injects them into Secrets/Routes; needed for route TLS after upgrade |
| 7.8 | ⬜ | CA in `user-ca-bundle` is **gr8it** (not socpoist!) | `oc get cm user-ca-bundle -n apc-crossplane-system -o yaml \| grep -A1 "gr8it"` | `gr8it-root` present | **Known recurring issue**: wrong CA causes x509 "unknown authority" on all Crossplane→Vault calls |

---

## 8. Networking — MetalLB, Ingress & Multus

| # | Status | Check | Command | Expected | Comment |
|---|--------|-------|---------|----------|---------|
| 8.1 | ⬜ | MetalLB speaker pods running | `oc get pods -n metallb-system` | All `Running` (speaker DaemonSet + controller) | MetalLB provides HCP spoke API VIP; if speaker dies, spokea1 control plane becomes unreachable |
| 8.2 | ⬜ | HCP API VIP assigned | `oc describe servicel2statuses -n metallb-system \| grep "10.11.60.42"` | VIP `10.11.60.42` assigned to a node | VIP assignment must survive hub node reboots during upgrade; verifies L2 advertisement healthy |
| 8.3 | ⬜ | HCP API VIP reachable | `curl -sk https://10.11.60.42:6443/healthz` | `ok` | Functional test of MetalLB + OVN path; unreachable = spokea1 cluster is already down |
| 8.4 | ⬜ | Ingress router pods — 3 replicas | `oc get pods -n openshift-ingress` | 3 pods `Running` | Ingress upgrade is rolling; < 3 replicas means single-router failure = full ingress outage |
| 8.5 | ⬜ | Ingress connectivity test | `curl -sIL https://vault.apps.huba.qa.sp.gr8it.cloud` | HTTP 200/301/302 | Baseline for post-upgrade comparison; confirms wildcard cert and router are working |
| 8.6 | ⬜ | EgressIP — all IPs assigned to nodes | `oc get egressip` | Every EgressIP has `ASSIGNED NODE` (huba CIDR: `10.11.61.0/24`) | External systems filter by source IP; unassigned EgressIP = unexpected source IP = connection rejection |
| 8.7 | ⬜ | OVN-Kubernetes pods healthy | `oc get pods -n openshift-ovn-kubernetes` | All `Running` | OVN-K is the CNI; unhealthy pods = network disruption during upgrade's pod restarts |
| 8.8 | ⬜ | Multus NNCP applied on worker nodes | `oc get nncp` | All `NodeNetworkConfigurationPolicy` in `Available` state | NNCP defines bond1, VLANs 1161/1162/1166 and bridge br-1166 needed by ODF and VMs |
| 8.9 | ⬜ | Multus bond1 + VLANs present on workers | `oc get nnce` | `Available` for qahuba-w01/w02/w03 (bond1, VLANs 1161/1162/1166, bridge br-1166) | Per-node enforcement status; NNCE failure = specific worker missing network config |
| 8.10 | ⬜ | NetworkAttachmentDefinitions present | `oc get net-attach-def -A` | `odf`, `vmlm`, `vm-br-1166` present | NADs are used by ODF pods (dedicated storage network VLAN 1161) and by KubeVirt VMs |
| 8.11 | ⬜ | Proxy reachable from cluster | `curl -x http://10.11.0.10:3128 -I https://registry.redhat.io` | HTTP 200/302 | OCP upgrade pulls images via proxy; unreachable proxy = image pull failure = upgrade stuck |

---

## 9. HCP / spokea1 (from hub perspective)

| # | Status | Check | Command | Expected | Comment |
|---|--------|-------|---------|----------|---------|
| 9.1 | ⬜ | HostedCluster `spokea1` available | `oc get hostedcluster spokea1 -n spokea1` | `Available=True`, no errors in `.status.conditions` | Hub upgrade reboots nodes hosting spokea1 control plane (NS `spokea1-spokea1`); must start healthy |
| 9.2 | ⬜ | NodePool not updating | `oc get nodepool -n spokea1` | `CURRENT NODES = DESIRED`, **no** `UPDATINGCONFIG` / `UPDATINGVERSION` | In-progress nodepool update = spoke worker reboots in flight; don't stack hub upgrade on top |
| 9.3 | ⬜ | HCP control plane pods healthy | `oc get pods -n spokea1-spokea1` | All `Running` / `Completed` | HCP control plane runs as regular pods on hub workers; hub node drains during upgrade evict them |
| 9.4 | ⬜ | No pod in CrashLoopBackOff in HCP NS | `oc get pods -n spokea1-spokea1 \| grep -v "Running\|Completed"` | Empty output | Any crashloop = spokea1 already partially degraded; hub upgrade will make it completely unavailable |
| 9.5 | ⬜ | ACM ManagedCluster `spokea1` available | `oc get managedcluster spokea1` | `Available=True` | ACM is used for policy propagation; disconnected spoke = policies won't apply post-upgrade |
| 9.6 | ⬜ | spokea1 worker nodes ready | `oc --context sp-spokea1 get nodes` | All workers `Ready` | Smoke-test that spoke is operationally healthy from its own perspective |
| 9.7 | ⬜ | spokea1 COs healthy | `oc --context sp-spokea1 get co` | All `Available=True, Degraded=False` | CO degradation on spoke before hub upgrade = spoke may be broken during hub upgrade window |

---

## 10. ACM (MultiClusterHub)

| # | Status | Check | Command | Expected | Comment |
|---|--------|-------|---------|----------|---------|
| 10.1 | ⬜ | MCH phase `Running` | `oc get multiclusterhub -n open-cluster-management` | `Running` | ACM MCH reconciles all hub-side ACM components; not Running = ACM management plane broken |
| 10.2 | ⬜ | ACM pods healthy | `oc get pods -n open-cluster-management \| grep -v "Running\|Completed"` | Empty output | ACM upgrade compatibility must be verified pre-OCP-upgrade (ACM has OCP version dependency matrix) |
| 10.3 | ⬜ | ManagedCluster `local-cluster` | `oc get managedcluster local-cluster` | `Available=True` | local-cluster represents hub itself; disconnected = ACM lost self-connection during upgrade |
| 10.4 | ⬜ | ManagedCluster `spokea1` | `oc get managedcluster spokea1` | `Available=True` | Duplicate of 9.5 — checked here to confirm ACM-specific connectivity |
| 10.5 | ⬜ | MCE (multicluster-engine) healthy | `oc get mce multiclusterengine` | `Available` phase | MCE is ACM's core engine; degraded MCE = HCP management, cluster discovery broken |
| 10.6 | ⬜ | MCO (MultiClusterObservability) healthy | `oc get multiclusterobservability observability -n open-cluster-management-observability` | `Ready` | MCO aggregates metrics from both clusters via Thanos; needed for upgrade monitoring dashboard |
| 10.7 | ⬜ | Thanos OBC for MCO bound | `oc get obc -n open-cluster-management-observability` | All OBCs `Bound` | MCO stores metrics in object storage; unbound OBC = metrics collection stops silently |

---

## 11. Quay Registry

| # | Status | Check | Command / UI | Expected | Comment |
|---|--------|-------|-------------|----------|---------|
| 11.1 | ⬜ | Quay pods running — **no CrashLoopBackOff** | `oc get pods -n quay` | All `Running`, especially `quay-app` pods | Known issue: startup probe failures cause HPA to scale to 20 pods → resource exhaustion cluster-wide |
| 11.2 | ⬜ | QuayRegistry CR `Available=True` | `oc get quayregistry apc-registry -n quay -o yaml \| grep -A3 conditions` | `type: Available, status: "True"` | Quay operator reconciles on every CR change; not-Available = reconciliation loop that stresses cluster |
| 11.3 | ⬜ | HPA replica count **not escalated** | `oc get hpa -n quay` | Replicas in range 1-3 (NOT 20!) | 20 replicas = known Quay failure mode (startup probe loop); upgrade on escalated HPA = resource crash |
| 11.4 | ⬜ | Quay app resource spec matches committed values | `oc get quayregistry apc-registry -n quay -o json \| jq '.spec.components[] \| select(.kind=="quay").overrides.resources'` | `requests.cpu=500m, requests.memory=8Gi, limits.cpu=4, limits.memory=10Gi` | P.1 issue: verifies file change actually applied to cluster (P.1 must be resolved first) |
| 11.5 | ⬜ | Quay CPU / Memory usage healthy | `oc adm top pods -n quay` | App pod below 3 CPU / 8Gi | High pre-upgrade resource usage = OOM kill risk when upgrade adds scheduling pressure |
| 11.6 | ⬜ | Quay UI accessible | `curl -sI https://apc-registry-quay-quay.apps.huba.qa.sp.gr8it.cloud` | HTTP 200 | Functional smoke test; also confirms Quay image serving (used by upgrade image pull) is working |
| 11.7 | ⬜ | Quay Postgres PVC bound (100Gi) | `oc get pvc -n quay` | PVC `Bound`, size `100Gi` | Lost PVC = entire Quay metadata DB gone; check before any operation involving storage changes |
| 11.8 | ⬜ | Quay OBC (ObjectBucketClaim) bound | `oc get obc -n quay` | `Bound` | OBC stores actual container image blobs; unbound = all image pushes/pulls fail |
| 11.9 | ⬜ | Quay startup/readiness probe passing | `oc describe pod -n quay -l quay-component=quay-app \| grep -A5 "Liveness\|Readiness"` | No probe failures in recent events | Failing probe = HPA will scale up (see 11.3); catch early before upgrade triggers pod restarts |
| 11.10 | ⬜ | OADP backup schedule `02-huba-quay` active | `oc get schedule -n openshift-adp \| grep quay` | Schedule exists, not suspended | Quay backup runs daily at 23:50; must be active before major change as last-known-good |
| 11.11 | ⬜ | Last Quay backup completed | `oc get backup -n openshift-adp \| grep quay` | Most recent `Completed` | Verifies backup ran successfully (hooks: pg_dump pre-backup); failed backup = unsafe to proceed |

---

## 12. ACS (StackRox)

| # | Status | Check | Command / UI | Expected | Comment |
|---|--------|-------|-------------|----------|---------|
| 12.1 | ⬜ | ACS Central pod running | `oc get pods -n stackrox \| grep central` | `Running` | ACS Central requires compatible OCP version; check running before upgrade to confirm no pre-existing issue |
| 12.2 | ⬜ | ACS Central DB pod running | `oc get pods -n stackrox \| grep central-db` | `Running` | Central DB is CNPG-based; DB pod failure = entire ACS loses persistence |
| 12.3 | ⬜ | ACS Central UI accessible | `curl -sk https://central-stackrox.apps.huba.qa.sp.gr8it.cloud:443 -o /dev/null -w "%{http_code}"` | `200` or `301` | Functional test; ACS routes must survive ingress controller upgrade |
| 12.4 | ⬜ | ACS Sensor on huba running | `oc get pods -n stackrox \| grep sensor` | `Running` | Sensor connects to Central; upgrade may break this connection temporarily — confirm healthy before |
| 12.5 | ⬜ | ACS Collector DaemonSet healthy | `oc get ds -n stackrox \| grep collector` | `DESIRED = READY` | Collector runs on every node; DaemonSet gets recreated during node upgrades — must start healthy |
| 12.6 | ⬜ | SecuredCluster CR status | `oc get securedcluster -n stackrox` | `Reconciled` / `Ready` | CR-level reconciliation status from the ACS operator |
| 12.7 | ⬜ | ACS InstallPlan manually approved | `oc get installplan -n rhacs-operator` | Approved (Manual approval is expected — verify it is not blocking) | ACS uses Manual OLM approval; an unapproved pending InstallPlan means operator won't upgrade automatically |
| 12.8 | ⬜ | **CNPG cluster (ACS DB) healthy** | `oc get cluster -n stackrox` | `STATUS=Cluster in healthy state`, `READY` | ACS Central DB runs on CNPG; cluster `Failed`/`Unknown` = ACS loses persistence on pod restart during upgrade |

---

## 13. Crossplane

| # | Status | Check | Command | Expected | Comment |
|---|--------|-------|---------|----------|---------|
| 13.1 | ⬜ | Crossplane pods running | `oc get pods -n apc-crossplane-system` | All `Running` | Crossplane manages Vault KV/Policy resources; if Crossplane breaks, secret configuration drifts |
| 13.2 | ⬜ | Vault Provider installed & healthy | `oc get provider -n apc-crossplane-system` | `Installed=True, Healthy=True` | Provider health = TLS + auth to Vault working; unhealthy = all Crossplane→Vault operations fail |
| 13.3 | ⬜ | ProviderConfig `vault-hub-provider-config` ready | `oc get providerconfig` | `Ready=True` | ProviderConfig holds Vault connection params; not ready = all managed resources in `ReconcileError` |
| 13.4 | ⬜ | **CA in `user-ca-bundle` is gr8it (CRITICAL)** | `oc get cm user-ca-bundle -n apc-crossplane-system -o yaml \| grep 'gr8it'` | `gr8it-root` present — **NOT** socpoist RootCA-Sp2 | **Known critical**: OCP upgrade can reset ConfigMaps from cluster config; re-check after upgrade too |
| 13.5 | ⬜ | No x509 errors in Crossplane logs | `oc logs -l app=crossplane -n apc-crossplane-system --tail=50` | No `x509: certificate signed by unknown authority` | Existing x509 errors = CA issue already present; upgrade will not fix it and may add more failures |

---

## 14. Operators (OLM)

| # | Status | Check | Command | Expected | Comment |
|---|--------|-------|---------|----------|---------|
| 14.1 | ⬜ | All CSVs `Succeeded` | `oc get csv -A \| grep -v Succeeded` | Empty (all Succeeded) | Non-Succeeded CSV = operator partially installed; OCP upgrade may change CRD versions and break it |
| 14.2 | ⬜ | No pending InstallPlans (except Manual) | `oc get installplan -A \| grep -v Complete` | Empty, or only Manual approvals that are expected | Pending auto-approve = operator trying to upgrade itself; racing with OCP upgrade causes conflicts |
| 14.3 | ⬜ | All Subscriptions `AtLatestKnown` | `oc get sub -A -o json \| jq '.items[] \| select(.status.state != "AtLatestKnown") \| {name:.metadata.name,ns:.metadata.namespace,state:.status.state}'` | Empty output | Not-at-latest = operator is pending upgrade; clarify if this is intentional before OCP upgrade |
| 14.4 | ⬜ | No BundleUnpacking failures | `oc get sub -A -o json \| jq '.items[] \| select(.status.conditions[]?.type=="BundleUnpacking") \| .metadata.name'` | Empty output | Bundle unpack failure = OLM can't retrieve operator image from registry; may indicate proxy issue |
| 14.5 | ⬜ | OLM operator logs clean | `oc logs -l app=olm-operator -n openshift-operator-lifecycle-manager --tail=50` | No ERROR / FATAL | OLM errors affect all operator management; upgrade adds significant OLM workload |
| 14.6 | ⬜ | **ACS operator — Manual installPlan** check | `oc get installplan -n rhacs-operator` | Acknowledge current state (Manual approval is intentional) | Manual approval is intentional; document state so it's clear what's pending post-upgrade |

---

## 15. Monitoring & Alerting

| # | Status | Check | Command / UI | Expected | Comment |
|---|--------|-------|-------------|----------|---------|
| 15.1 | ⬜ | Prometheus pods running | `oc get pods -n openshift-monitoring \| grep prometheus` | All `Running` | Monitoring must be healthy to detect upgrade-induced failures in real time |
| 15.2 | ⬜ | Alertmanager pods running | `oc get pods -n openshift-monitoring \| grep alertmanager` | All `Running` | Alert routing to external receivers required; silent upgrade = missed incidents |
| 15.3 | ⬜ | No firing **Critical** alerts | OCP Console → Observe → Alerting | Zero Critical alerts (document any Warning alerts) | Firing Critical alert = active problem; starting upgrade on top risks simultaneous failures |
| 15.4 | ⬜ | UWM Prometheus running | `oc get pods -n openshift-user-workload-monitoring` | All `Running` | UWM collects app-level metrics (Vault, Quay, etc.); must survive upgrade |
| 15.5 | ⬜ | Grafana pod running | `oc get pods -n apc-observability \| grep grafana` | `Running` | Used for upgrade monitoring dashboards (availability reports, MCO dashboards) |
| 15.6 | ⬜ | Blackbox exporter pods | `oc get pods -n apc-blackbox-exporter` | `Running` | Probes external endpoints; confirms network paths before upgrade |
| 15.7 | ⬜ | OADP backup schedules active | `oc get schedule -n openshift-adp` | `01-huba-prometheus`, `02-huba-quay`, `03-huba-xca-vm` present, not suspended | Three backup schedules must be active; suspended = no pre-upgrade backup will run automatically |
| 15.8 | ⬜ | Last backups successful | `oc get backup -n openshift-adp --sort-by=.metadata.creationTimestamp \| tail -5` | All `Completed` | Confirms backup pipeline end-to-end (OADP → hooks → Velero → NooBaa S3) is working |
| 15.9 | ⬜ | DataProtectionApplication healthy | `oc get dpa -n openshift-adp` | `Reconciled` | DPA configures BSL/VSL pointing to NooBaa; not reconciled = backups won't run |
| 15.10 | ⬜ | S3 bucket reachable (Noobaa/OBC) | Check OADP backup logs for connection errors | No S3 connection errors | NooBaa S3 is the backup target; connection error = silent backup failure |
| 15.11 | ⬜ | **BSL (BackupStorageLocation) Available** | `oc get backupstoragelocation -n openshift-adp` | `PHASE=Available` for all BSLs | If BSL is not `Available`, OADP cannot create a backup before the upgrade — DPA may be OK but BSL may not |
| 15.12 | ⬜ | **Alertmanager receiver configured** | `curl -sk http://localhost:9093/api/v2/receivers` (via port-forward) | At least one receiver other than `null` / empty | AM may be running but have only null receiver = no alerting notifications during upgrade; silent upgrade = missed incident |

---

## 16. Logging (COO / Loki)

| # | Status | Check | Command | Expected | Comment |
|---|--------|-------|---------|----------|---------|
| 16.1 | ⬜ | Logging namespace pods running | `oc get pods -n openshift-logging` | All `Running` | Log collection must be running to capture upgrade events for post-mortem if needed |
| 16.2 | ⬜ | LokiStack healthy | `oc get lokistack -n openshift-logging` | `Ready` | Loki is the log backend; not Ready = logs not being stored, blind during upgrade |
| 16.3 | ⬜ | ClusterLogForwarder healthy | `oc get clusterlogforwarder -n openshift-logging` | `Ready` | CLF routes logs to Loki; not healthy = log pipeline broken before upgrade |
| 16.4 | ⬜ | COO (Cluster Observability Operator) | `oc get pods -n openshift-operators-redhat \| grep cluster-observability` | `Running` | COO manages LokiStack and UIPlugin lifecycle; upgrade may touch this operator |
| 16.5 | ⬜ | NetObserv FlowCollector healthy | `oc get flowcollector cluster` | `Ready` (condition check) | eBPF-based network flow collection; FlowCollector CR drives DaemonSet on every node |
| 16.6 | ⬜ | NetObserv LokiStack ready | `oc get lokistack netobserv-loki -n apc-netobserv` | `Ready` | Separate LokiStack from main logging stack; stores network flow data for post-upgrade analysis |
| 16.7 | ⬜ | NetObserv pods running | `oc get pods -n apc-netobserv \| grep -v "Running\|Completed"` | Empty output | NetObserv agent runs as DaemonSet; node upgrades will trigger pod restarts — confirm pre-upgrade state |

---

## 17. Tracing (Tempo)

| # | Status | Check | Command | Expected | Comment |
|---|--------|-------|---------|----------|---------|
| 17.1 | ⬜ | Tempo pods running | `oc get pods -n apc-observability \| grep tempo` | `Running` | Tracing backend for distributed traces; recently added — verify stable before upgrade stresses cluster |
| 17.2 | ⬜ | TempoStack / TempoMonolithic CR | `oc get tempostack -A 2>/dev/null; oc get tempomonolithic -A 2>/dev/null` | `Ready` (recently added — verify exists) | CR existence confirms operator has reconciled the deployment successfully |
| 17.3 | ⬜ | OpenTelemetry Collector pods (spokea1) | `oc --context sp-spokea1 get pods -n apc-observability \| grep otel` | `Running` | OTEL collector forwards traces from spoke; spoke upgrade is separate but confirm cross-cluster path works |

---

## 18. Kyverno

| # | Status | Check | Command | Expected | Comment |
|---|--------|-------|---------|----------|---------|
| 18.1 | ⬜ | Kyverno pods running | `oc get pods -n apc-kyverno` | All `Running` | Kyverno is admission webhook; if down, new/updated objects fail with webhook timeout |
| 18.2 | ⬜ | All ClusterPolicies ready | `oc get cpol` | `READY=True` for all policies | Not-ready policy = admission webhook returns error for matching resources; upgrade generates many new objects |
| 18.3 | ⬜ | No blocking policy violations | `oc get events -A --field-selector reason=PolicyViolation \| tail -20` | No blocking events against running workloads | Existing violations = admission block; upgrade-created pods would be rejected if they match the policy |
| 18.4 | ⬜ | OBC allinfo Kyverno policy active | `oc get cpol generate-bucket-secret-allinfo` | `READY=True` | Policy generates S3 secret for all new OBCs; required for backup OBCs created during/after upgrade |
| 18.5 | ⬜ | `reportsController` disabled on huba | `oc get deploy -n apc-kyverno \| grep reports` | Deployment NOT present (intentional — ACM CRD conflict workaround) | Intentionally disabled due to ACM CRD conflict; verify upgrade didn't re-enable it |

---

## 19. Compliance Operator

| # | Status | Check | Command | Expected | Comment |
|---|--------|-------|---------|----------|---------|
| 19.1 | ⬜ | Compliance operator pod running | `oc get pods -n openshift-compliance` | `Running` | Compliance scans run post-upgrade; operator must be healthy for scan scheduling |
| 19.2 | ⬜ | ScanSettingBindings exist | `oc get scansettingbinding -n openshift-compliance` | Bindings present | SSBs schedule periodic CIS scans; if deleted, compliance posture is unmeasured post-upgrade |
| 19.3 | ⬜ | No compliance scan in Error/Failed | `oc get compliancescan -n openshift-compliance` | No `PHASE=Error` | Error scan before upgrade = existing compliance issue that should be documented before audit trail |

---

## 20. Group Sync Operator

| # | Status | Check | Command | Expected | Comment |
|---|--------|-------|---------|----------|---------|
| 20.1 | ⬜ | Group Sync Operator pod running | `oc get pods -n group-sync-operator` | `Running` | Group sync must work during upgrade for admin access via AD/LDAP groups |
| 20.2 | ⬜ | GroupSync CRs configured | `oc get groupsync -n group-sync-operator` | GroupSync objects exist | CR existence confirms sync configuration hasn't been accidentally deleted |
| 20.3 | ⬜ | Last sync successful | `oc describe groupsync -n group-sync-operator \| grep -A3 conditions` | No error conditions | Failed sync = OCP groups stale = upgrade team may not have cluster-admin during upgrade window |
| 20.4 | ⬜ | Expected OCP groups exist | `oc get group \| grep -E "ocp_admins\|apc_operator"` | Groups `ocp_admins_huba`, `ocp_admins_spokea`, `apc_operator_huba` present with members | Verifies admin groups populated; if groups empty, cluster-admin access during upgrade depends on kubeconfig only |

---

## 21. Additional Components

| # | Status | Check | Command | Expected | Comment |
|---|--------|-------|---------|----------|---------|
| 21.1 | ⬜ | apc-warmcache pod running | `oc get pods -n apc-warmcache` | `Running` | Warmcache pre-pulls images on nodes; if down before upgrade, first-start latency for apps increases |
| 21.2 | ⬜ | Stakater Reloader running | `oc get pods -n apc-stakater-reloader` | `Running` | Reloader watches Secrets/ConfigMaps and restarts pods; needed post-upgrade to propagate cert changes |
| 21.3 | ⬜ | EgressIP management controller | `oc get pods -n apc-egressip-management` | `Running` | Controller re-assigns EgressIPs when nodes go NotReady; critical during upgrade's rolling node drain |
| 21.4 | ⬜ | CNPG operator running | `oc get pods -n apc-cnpg-operator` | `Running` | CNPG manages PostgreSQL for ACS; operator must be running to handle failovers during upgrade |
| 21.5 | ⬜ | Grafana operator running | `oc get pods -n apc-grafana-operator` | `Running` | Grafana instance managed by operator; operator crash during upgrade leaves dashboards without reconciliation |
| 21.6 | ⬜ | External Secrets operator running | `oc get pods -n apc-external-secrets-operator` | `Running` | ESO refreshes Vault secrets; if down, secrets in namespaces become stale after pod restarts |

---

## 22. OpenShift Virtualization (KubeVirt/CNV)

| # | Status | Check | Command | Expected | Comment |
|---|--------|-------|---------|----------|---------|
| 22.1 | ⬜ | HyperConverged CR available | `oc get hco kubevirt-hyperconverged -n openshift-cnv` | `Available=True, Progressing=False, Degraded=False` | HCO is the root CNV resource; degraded = KubeVirt or CDI broken; VMs will fail to start |
| 22.2 | ⬜ | KubeVirt operator pods running | `oc get pods -n openshift-cnv \| grep virt-operator` | `Running` (2 replicas) | virt-operator manages KubeVirt lifecycle; down = no VM lifecycle operations possible |
| 22.3 | ⬜ | CDI (Containerized Data Importer) running | `oc get pods -n openshift-cnv \| grep cdi` | All `Running` | CDI handles VM disk (DataVolume) operations; not running = disk clone/import fails |
| 22.4 | ⬜ | XCA VM running (`xca` namespace) | `oc get vm -n xca` | `READY=True`, phase `Running` | XCA is the certificate authority VM; if down, TLS cert issuance for infra services breaks |
| 22.5 | ⬜ | XCA VM instance alive | `oc get vmi -n xca` | VMI present, `Running` | VMI = actual running instance; VM=Ready but no VMI = VM not actually running (boot failure) |
| 22.6 | ⬜ | Veeam proxy VM running (`veeam` namespace) | `oc get vm -n veeam` | `READY=True`, phase `Running` | Veeam proxy is the backup agent for physical-to-OCP backups; down = backup jobs fail |
| 22.7 | ⬜ | VM live-migrate network (vmlm) functional | `oc get net-attach-def vmlm -n openshift-cnv` | Exists | VLAN 1162 macvlan for VM live migration; must exist before upgrade triggers any node drain |
| 22.8 | ⬜ | No VM in error state | `oc get vmi -A \| grep -v "Running\|Succeeded"` | Empty output | VM errors indicate KubeVirt or storage problems; upgrade-induced node drain forces VM migration |
| 22.9 | ⬜ | **VM live-migration capability** | `oc get vmi -A -o json \| jq '.items[] \| {name:.metadata.name,ns:.metadata.namespace,migrationMethod:.status.migrationMethod}'` | `LiveMigration` (not `BlockMigration`) | `BlockMigration` = storage does not support live migration → VM will be **evicted (shut down)** on node drain during upgrade; XCA and Veeam VMs must be live-migratable |

---

## 23. Bastions & External Services (LDAP / Transit Vault)

| # | Status | Check | Command | Expected | Comment |
|---|--------|-------|---------|----------|---------|
| 23.1 | ⬜ | Bastion hosts reachable | `ping -c2 10.11.70.11; ping -c2 10.11.70.12` | Both respond | Bastions host LDAP and Transit Vault VIPs via keepalived; unreachable = no admin auth during upgrade |
| 23.2 | ⬜ | LDAP VIP (keepalived) active | `ldapsearch -x -H ldaps://10.11.70.30:636 -b "" -s base` | Response from LDAP | LDAP VIP used for OCP OAuth; down = no AD/LDAP login; only kubeconfig-based access survives |
| 23.3 | ⬜ | Keepalived running on both bastions | SSH → bjc1 + bjc2: `systemctl status keepalived` | `active (running)` on both | Single-bastion keepalived = VIP failover broken; if that bastion goes down, auth dies completely |
| 23.4 | ⬜ | Transit Vault VIP accessible | `curl -sk https://10.11.70.31:8200/v1/sys/health \| jq .sealed` | `false` | Transit Vault = auto-unseal backend for huba Vault; if unreachable, any Vault pod restart = permanent seal |
| 23.5 | ⬜ | AD OAuth login works | OCP Console → AD identity provider | Successful login | End-to-end auth test; confirms LDAP → OCP OAuth → group binding works for upgrade operator |
| 23.6 | ⬜ | LDAP OAuth login works | OCP Console → LDAP identity provider | Successful login | Second identity provider; both must work to confirm no authentication regression |
| 23.7 | ⬜ | AD group sync working | `oc get group \| grep ocp_admins_huba` | Group exists and has members | Final auth gate; missing members = cluster-admin access depends on static kubeconfig only |

---

## 24. GitOps Repo Consistency

| # | Status | Check | Command | Expected | Comment |
|---|--------|-------|---------|----------|---------|
| 24.1 | ⬜ | `rendered/` matches current values | `make render ENV=huba && git diff rendered/` | No diff | Stale rendered manifests = ArgoCD auto-sync applies outdated config during/after upgrade |
| 24.2 | ⬜ | Git clean (including P.1 resolved) | `git status` | `working tree clean` | Required for reproducible rollback; uncommitted changes = recovery state is unknown |
| 24.3 | ⬜ | `kubeVersion` in global values | `grep kubeVersion gitops/environments/__global/values.yaml` | `v1.30` — update to match **target** OCP version before `make render` if needed | Some Helm charts use `.Capabilities.KubeVersion` for conditional rendering; wrong version = wrong manifests |
| 24.4 | ⬜ | All deployed chart versions match `versions.yaml.gotmpl` | ArgoCD apps vs. file | No discrepancies | Discrepancy = manual change applied to cluster outside GitOps; must be reconciled before upgrade |
| 24.5 | ⬜ | No ArgoCD app in `Unknown` health | `oc get apps -n apc-gitops -o json \| jq '.items[] \| select(.status.health.status == "Unknown") \| .metadata.name'` | Empty output | Unknown health = ArgoCD can't determine state; could be CRD missing or API server issue |

---

## 25. Pre-Upgrade State Snapshot

Run these commands and **save the output** before starting the upgrade for post-upgrade comparison:

```bash
# 1. OCP version
oc version > pre-upgrade-state.txt

# 2. All Cluster Operators
oc get co -o wide >> pre-upgrade-state.txt

# 3. All nodes + versions
oc get nodes -o wide >> pre-upgrade-state.txt

# 4. All CSVs (operators)
oc get csv -A >> pre-upgrade-state.txt

# 5. ETCD health (from etcd pod)
etcdctl endpoint status --cluster -w table >> pre-upgrade-state.txt

# 6. Ceph status (from rook-ceph-tools)
ceph status >> pre-upgrade-state.txt
ceph osd df >> pre-upgrade-state.txt

# 7. Vault status
oc exec -n apc-vault vault-0 -- vault status >> pre-upgrade-state.txt
oc exec -n apc-vault vault-0 -- vault operator raft list-peers >> pre-upgrade-state.txt

# 8. ArgoCD apps
oc get apps -n apc-gitops >> pre-upgrade-state.txt

# 9. HCP spoke state
oc get hostedcluster -n spokea1 >> pre-upgrade-state.txt
oc get nodepool -n spokea1 >> pre-upgrade-state.txt

# 10. Active alerts (via port-forward)
kubectl port-forward svc/alertmanager-operated -n openshift-monitoring 9093 &
curl -s http://localhost:9093/api/v2/alerts | jq '.[].labels' >> pre-upgrade-state.txt

# 11. MachineConfigPools
oc get mcp >> pre-upgrade-state.txt

# 12. VirtualMachines (KubeVirt)
oc get vm -A >> pre-upgrade-state.txt
oc get vmi -A >> pre-upgrade-state.txt

# 13. MD RAID state on all masters (run per master)
for m in qahuba-m01 qahuba-m02 qahuba-m03; do
  echo "=== $m ===" >> pre-upgrade-state.txt
  ssh core@$m "cat /proc/mdstat" >> pre-upgrade-state.txt
done
```

---

## 🚫 Upgrade BLOCKERS — Do NOT proceed if any of these are true:

- [ ] **Target OCP version NOT visible in `oc adm upgrade`** (upgrade graph unavailable)
- [ ] **PDB exists with `ALLOWED DISRUPTIONS=0`** — node drain will be blocked
- [ ] **Deprecated/removed API actively used** by workloads or operators
- [ ] Any node is `NotReady`
- [ ] Any MachineConfigPool is `Updating=True`, `Degraded=True`, or `Paused=True`
- [ ] ETCD cluster is not healthy (not all 3 members `healthy`)
- [ ] **ETCD DB size > 500 MB** without prior defragmentation
- [ ] Any Cluster Operator is `Degraded=True`
- [ ] ODF/Ceph is `HEALTH_ERR` (WARN is a judgment call — document and decide)
- [ ] Vault is `Sealed` on any replica
- [ ] Transit unseal Vault (`vault.comm.qa.sp.gr8it.cloud`) is unreachable
- [ ] ArgoCD has infra-critical apps `OutOfSync` (non-infra apps: note and decide)
- [ ] Active **Critical** Prometheus alerts firing (infra-related)
- [ ] **Alertmanager has no functional receiver configured** (null receiver only)
- [ ] Ceph `full_ratio` was manually modified (prior OSD_FULL condition)
- [ ] HostedCluster `spokea1` is not `Available=True`
- [ ] `rendered/` directory has uncommitted diff (`git diff rendered/` non-empty)
- [ ] QuayRegistry HPA has escalated to maximum replicas (startup probe loop)
- [ ] `ocp-qahuba01/quay/02-quay-registry.yaml` changes (P.1) are **not committed and applied**
- [ ] `/var/lib/etcd` mount not `active` on any master (ETCD writing to root tmpfs)
- [ ] MD RAID degraded on any master node (LVM/TopoLVM backing ETCD volumes)
- [ ] XCA VM (`xca/sr-ba-xapc1xca-p11`) not `Running` (certificate infrastructure unavailable)
- [ ] **XCA or Veeam VM has `BlockMigration`** (VM will be evicted on node drain)
- [ ] `user-ca-bundle` in `apc-crossplane-system` does not contain `gr8it-root` CA
- [ ] **BSL (BackupStorageLocation) is not `Available`** — OADP cannot create backup
- [ ] **CNPG cluster for ACS is not `healthy`** (`oc get cluster -n stackrox`)

---

## 📋 Quick Health Commands

```bash
# Full health overview — nodes, COs, unhealthy pods
oc get nodes && echo "---" && oc get co && echo "---" && \
  oc get pods -A --field-selector=status.phase!=Running | grep -v "Completed\|Succeeded"

# Count non-Running/Completed pods
oc get pods -A --no-headers | grep -v "Running\|Completed\|Succeeded" | wc -l

# ArgoCD apps not Healthy or Synced
oc get apps -n apc-gitops -o json | jq '.items[] |
  select(.status.health.status != "Healthy" or .status.sync.status != "Synced") |
  {name:.metadata.name, health:.status.health.status, sync:.status.sync.status}'

# Certificates expiring within 30 days
oc get cert -A -o json | jq --argjson now $(date +%s) '
  .items[] | select(.status.notAfter != null) |
  select((.status.notAfter | fromdateiso8601) < ($now + 2592000)) |
  {name:.metadata.name, ns:.metadata.namespace, expiry:.status.notAfter}'

# EgressIP assignment check
oc get egressip -o wide

# MetalLB VIP assignment
oc describe servicel2statuses -n metallb-system

# All OLM CSVs not Succeeded
oc get csv -A | grep -v Succeeded

# MachineConfigPool status
oc get mcp -o wide

# MD RAID status on masters (requires SSH or debug pod)
for m in qahuba-m01 qahuba-m02 qahuba-m03; do echo "=== $m ==="; ssh core@$m cat /proc/mdstat; done

# VirtualMachine status + migration method
oc get vm,vmi -A
oc get vmi -A -o json | jq '.items[] | {name:.metadata.name,ns:.metadata.namespace,migrationMethod:.status.migrationMethod}'

# ETCD LVM logical volumes
oc get logicalvolume -l component=lvm-etcd

# Upgrade path + channel
oc adm upgrade
oc get clusterversion -o json | jq .spec.channel

# PodDisruptionBudgets blocking drain
oc get pdb -A -o json | jq '.items[] | select(.status.disruptionsAllowed==0) | {name:.metadata.name,ns:.metadata.namespace}'

# ETCD DB size
oc -n openshift-etcd exec $(oc get pods -n openshift-etcd -l app=etcd -o name | head -1) -- \
  etcdctl endpoint status --cluster -w table

# BackupStorageLocation status
oc get backupstoragelocation -n openshift-adp

# CNPG cluster (ACS DB) health
oc get cluster -n stackrox

# Alertmanager receivers (via port-forward)
kubectl port-forward svc/alertmanager-operated -n openshift-monitoring 9093 &
curl -s http://localhost:9093/api/v2/receivers | jq '.[].name'
```

---

*Checklist created: 2026-03-17 | Updated: 2026-03-17 (added: upgrade path, PDB, deprecated API, ETCD DB size, CNPG, BSL, AM receiver, VM live-migration, operator compatibility matrix) | Cluster: huba (conf-sp-qa)*
