# rh-exam-agent-sno AgnosticD Config

Deploys **Single-Node OpenShift (SNO)** with the complete **rh-exam-agent** stack for Red Hat certification exam preparation.

## Architecture

**Components Deployed**:
- OpenShift 4.22 (Single-Node)
- Red Hat OpenShift AI (RHOAI 2.16)
- Red Hat Advanced Cluster Management (RHACM 2.17)
- OpenShift Virtualization (with nested virt support)
- rh-exam-agent application (FastAPI, React, LlamaStack, RAG pipeline, MCP server)

## Instance Type Variants

### 1. Nested Virtualization (Default - Cost-Optimized)

**Recommended for**: Development, testing, home lab validation

**Instance Types**:
- `m8i.4xlarge`: 16 vCPU, 64 GiB RAM, ~$2.50/hr
- `c8i.4xlarge`: 16 vCPU, 32 GiB RAM, ~$2.20/hr

**Features**:
- AWS nested virtualization (Feb 2026 feature)
- OpenShift Virt runs via nested KVM
- ~60% cost savings vs bare metal

### 2. Bare Metal (Performance-Optimized)

**Recommended for**: Production, GPU-intensive workloads

**Instance Type**:
- `g4dn.metal`: 96 vCPU, 384 GiB RAM, 8x NVIDIA T4 GPUs, $7.824/hr

**Features**:
- Native bare metal performance
- Hardware GPU acceleration
- No nested virt overhead

## Usage

### Provision SNO Cluster

```bash
cd ~/Development/agnosticd-v2

# Nested virt (default)
./bin/agd provision \
  -g rh-exam-sno-dev \
  -c rh-exam-agent-sno \
  --vars ~/Development/agnosticd-v2-vars/rh-exam-agent-sno/nested.yaml \
  -a sandbox1234

# Bare metal
./bin/agd provision \
  -g rh-exam-sno-prod \
  -c rh-exam-agent-sno \
  --vars ~/Development/agnosticd-v2-vars/rh-exam-agent-sno/baremetal.yaml \
  -a sandbox1234
```

### Cost Control (Stop/Start)

```bash
# Stop cluster (saves $$ when not in use)
./bin/agd stop -g rh-exam-sno-dev -c rh-exam-agent-sno -a sandbox1234

# Start cluster
./bin/agd start -g rh-exam-sno-dev -c rh-exam-agent-sno -a sandbox1234

# Check status
./bin/agd status -g rh-exam-sno-dev -c rh-exam-agent-sno -a sandbox1234
```

### Teardown

```bash
./bin/agd destroy -g rh-exam-sno-dev -c rh-exam-agent-sno -a sandbox1234
```

## Workloads

Deployed in order:

1. **cert-manager** (`agnosticd.core_workloads.ocp4_workload_cert_manager`) - Certificate management
2. **OpenShift AI** (`ocp4_workload_openshift_ai`) - RHOAI, GPU operator, Pipelines, Serverless, MinIO
3. **RHACM** (`ocp4_workload_rhacm`) - Cluster management for exam lab provisioning
4. **OpenShift Virt** (`ocp4_workload_openshift_virt`) - VM provisioning for RHEL exams
5. **rh-exam-agent** (`ocp4_workload_rh_exam_agent`) - Application stack via Helm/ArgoCD

## Verification

After provisioning:

```bash
# SSH to bastion
ssh lab-user@<bastion-ip>

# Check SNO node
oc get nodes

# Verify nested virt (if C8i/M8i/R8i)
oc get nodes -o jsonpath='{.items[*].status.allocatable.devices\.kubevirt\.io/kvm}'

# Check operators
oc get csv -A | grep -E 'rhods|rhacm|kubevirt|gpu-operator|pipelines'

# Check rh-exam-agent app
oc get pods -n rh-exam-agent
oc get route -n rh-exam-agent rh-exam-agent -o jsonpath='{.spec.host}'
```

## Configuration Variables

Override in your vars file (`agnosticd-v2-vars/rh-exam-agent-sno/*.yaml`):

```yaml
# Instance type
control_plane_instance_type: "m8i.4xlarge"  # or "g4dn.metal"
enable_nested_virtualization: true          # false for bare metal

# OpenShift version
host_ocp4_installer_version: "4.22"

# Component versions
ocp4_workload_openshift_ai_version: "2.16"
ocp4_workload_rhacm_version: "2.17"

# Application settings
ocp4_workload_rh_exam_agent_llm_model: "llama3.2-3b"  # or "granite-3.1-8b"
```

## Cost Estimates

**Nested Virt (m8i.4xlarge)**:
- $2.50/hr × 8 hours/day = **$20/day**
- Use stop/start for cost control

**Bare Metal (g4dn.metal)**:
- $7.824/hr × 8 hours/day = **$62.59/day**

## Troubleshooting

See [AgnosticD v2 Documentation](https://github.com/agnosticd/agnosticd-v2/tree/main/docs)

## Related ADRs

- **ADR-001**: Cluster Awareness & Lab Provisioning
- **ADR-005**: GitOps Deployment via ArgoCD
- **ADR-007**: AgnosticD Deployment Decision (TBD)
