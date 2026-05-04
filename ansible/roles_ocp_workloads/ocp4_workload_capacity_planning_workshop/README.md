# Strategic Capacity Planning Workshop - AgnosticD Workload

## Overview

This workload deploys a comprehensive Strategic Capacity Planning & Forecasting workshop for OpenShift at Scale. The workshop teaches platform engineers and SREs how to transition from reactive "firefighting" to predictive, data-driven capacity management.

**Workshop Duration:** 8 hours (full-day workshop)

**Technologies:**
- OpenShift Container Platform 4.14-4.19+
- Red Hat Advanced Cluster Management (RHACM) 2.14+
- OpenShift Virtualization (for multi-user mode)
- Showroom lab guide system
- Prometheus/Thanos/Grafana for observability

## Deployment Modes

### Single-User Mode (Default)
**Use Case:** Instructor-led demonstrations, individual learners

- **Resources:** Deploys to hub cluster namespace
- **User Count:** 1 (instructor or single learner)
- **Components:** Showroom, sample apps, terminal on hub cluster
- **RHACM:** Hub cluster with observability enabled

**Deployment:**
```bash
./bin/agd provision --config capacity_planning_workshop --vars /path/to/vars.yml
```

**vars.yml:**
```yaml
# Single-user mode (omit num_users or set to 0)
guid: abc123
cloud_provider: aws
# num_users: 0  # Optional - defaults to 0
```

---

### Multi-User Mode (SNO VMs via OpenShift Virtualization)
**Use Case:** Hands-on workshops with 1-3 concurrent users

- **Resources:** Each user gets dedicated SNO cluster (8 vCPU, 16GB RAM, 120GB storage)
- **User Count:** 1-3 users (configurable, resource-dependent)
- **Components:** Showroom on hub, sample apps on each SNO cluster
- **RHACM:** Hub-and-spoke architecture with unified observability

**Architecture:**
```
Hub Cluster (existing RHDP cluster)
  ├── RHACM MultiClusterHub
  ├── OpenShift Virtualization (CNV)
  ├── Showroom (per-user instances)
  └── ClusterPool (manages SNO VMs)
      ├── SNO VM 1 (user1) - 8 vCPU, 16GB RAM
      ├── SNO VM 2 (user2) - 8 vCPU, 16GB RAM
      └── SNO VM 3 (user3) - 8 vCPU, 16GB RAM
```

**Deployment:**
```bash
./bin/agd provision --config capacity_planning_workshop --vars /path/to/vars.yml
```

**vars.yml:**
```yaml
guid: abc123
cloud_provider: aws
num_users: 3  # Deploy 3 SNO VMs for 3 workshop users
```

---

## Resource Planning

### Single-User Mode Resource Requirements

| Component | CPU Request | Memory Request | Storage |
|-----------|-------------|----------------|---------|
| Showroom | 500m | 512Mi | - |
| Sample Apps (6 deployments) | ~1200m | ~1.5Gi | - |
| Terminal | 200m | 256Mi | 5Gi PVC |
| **Total** | **~2 cores** | **~2.5Gi** | **5Gi** |

**Recommended Hub Cluster:** 3 workers × 8 cores × 32GB

---

### Multi-User Mode Resource Requirements (SNO VMs)

**Per-User SNO VM:**
- **vCPU:** 8 cores (SNO minimum requirement)
- **Memory:** 16GB RAM (SNO minimum requirement)
- **Storage:** 120GB disk
- **Virtualization Overhead:** ~10% CPU, ~2GB RAM

**Effective Resource Cost per User:**
- **CPU:** ~8.8 cores (8 + 10% overhead)
- **Memory:** ~18GB (16GB + 2GB overhead)
- **Storage:** 120GB

**Maximum Users by Hub Cluster Size:**

| Hub Cluster | Total Cores | Total RAM | Max SNO VMs | Reasoning |
|-------------|-------------|-----------|-------------|-----------|
| 3 workers (16 cores, 32GB) | 48 cores | 96GB | **2 users** | Memory bottleneck: 96GB ÷ 18GB = 5.3 users theoretical, recommend 2 for safety |
| 5 workers (16 cores, 32GB) | 80 cores | 160GB | **3 users** | Balanced: 80 cores ÷ 8.8 = 9 users, 160GB ÷ 18GB = 8.8 users, recommend 3 with headroom |
| 10 workers (16 cores, 32GB) | 160 cores | 320GB | **5 users** | Theoretical max 17 users, but recommend 5 for stability and hub services |

**Note:** The workload enforces a maximum of 3 users by default (configurable via `ocp4_workload_capacity_planning_workshop_max_users`).

---

## Configuration Variables

See `defaults/main.yml` for complete list. Key variables:

### Core Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `ocp4_workload_capacity_planning_workshop_namespace` | `capacity-workshop` | Workshop namespace |
| `ocp4_workload_capacity_planning_workshop_helm_repo` | GitHub URL | Helm chart repository |
| `ocp4_workload_capacity_planning_workshop_content_repo` | GitHub URL | Lab guide content |

### Multi-User Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `num_users` | `0` | Number of workshop users (0 = single-user, 1+ = multi-user with SNO VMs) |
| `ocp4_workload_capacity_planning_workshop_max_users` | `3` | Maximum allowed users |
| `ocp4_workload_capacity_planning_workshop_sno_vcpu` | `8` | vCPU per SNO VM |
| `ocp4_workload_capacity_planning_workshop_sno_memory` | `16Gi` | RAM per SNO VM |
| `ocp4_workload_capacity_planning_workshop_sno_disk_size` | `120Gi` | Disk size per SNO VM |

---

## Deployment Timeline

**Single-User Mode:**
- RHACM deployment: ~5-10 minutes
- Workshop components via ArgoCD: ~3-5 minutes
- **Total:** ~15 minutes

**Multi-User Mode (3 users):**
- RHACM deployment: ~5-10 minutes
- OpenShift Virtualization installation: ~10-15 minutes
- SNO VM provisioning: ~5 minutes
- SNO cluster installation: **~30-45 minutes per cluster** (parallelized)
- Workshop components: ~3-5 minutes per cluster
- **Total:** ~60-75 minutes

**Critical:** Multi-user mode requires ~1 hour. Plan workshop start time accordingly.

---

## Troubleshooting

### SNO VM Provisioning Stuck

**Diagnosis:**
```bash
# Check VM status
oc get vm -n sno-workshop-pool

# Check Agent discovery
oc get agents -n sno-workshop-pool
```

**Resolution:**
```bash
# Restart stuck VMs
oc delete vmi -n sno-workshop-pool --all
```

### Insufficient Resources

**Diagnosis:**
```bash
# Check node capacity
oc get nodes -o custom-columns=NAME:.metadata.name,CPU:.status.allocatable.cpu,MEMORY:.status.allocatable.memory
```

**Resolution:**
- Reduce `num_users` to match available capacity
- Scale up hub cluster (add worker nodes)

---

## Cleanup

```bash
./bin/agd destroy --config capacity_planning_workshop --vars /path/to/vars.yml
```

**Manual cleanup (if destroy fails):**
```bash
# Release ClusterClaims
oc delete clusterclaim -l demo.redhat.com/guid=<guid> -n sno-workshop-pool

# Delete SNO VMs
oc delete vm -l demo.redhat.com/guid=<guid> -n sno-workshop-pool

# Delete namespaces
oc delete namespace sno-workshop-pool capacity-workshop
```

---

## Support

- **Workshop Content:** https://github.com/tosin2013/capacity-planning-lab-guide
- **Helm Charts:** https://github.com/tosin2013/openshift-capacity-planning-workshop

## Author

Tosin Akinosho (takinosh@redhat.com)
