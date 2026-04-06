# ocp4_workload_capacity_planning_workshop

Deploys the Strategic Capacity Planning & Forecasting Workshop for OpenShift.

## Overview

This workload deploys a comprehensive full-day workshop covering:
- Capacity baseline auditing
- Pod velocity forecasting with PromQL
- QoS classes and resource right-sizing
- Node density tuning
- RHACM multi-cluster observability
- Live chaos simulation (Black Friday scenario)
- 12-month strategic capacity roadmapping

## Prerequisites

- OpenShift 4.14+ cluster
- OpenShift GitOps (ArgoCD) operator installed
- Storage class available for RHACM observability (default: ODF Ceph RBD)
- Cluster admin privileges

## Variables

See `defaults/main.yml` for all configurable variables.

Key variables:
- `ocp4_workload_capacity_planning_workshop_namespace`: Workshop namespace (default: `capacity-workshop`)
- `ocp4_workload_capacity_planning_workshop_deploy_rhacm`: Deploy RHACM (default: `true`)
- `ocp4_workload_capacity_planning_workshop_helm_repo`: Helm chart repository URL
- `ocp4_workload_capacity_planning_workshop_content_repo`: Showroom lab guide repository URL

## Deployment via AgnosticD

```yaml
# In your vars file
workloads:
  - ocp4_workload_capacity_planning_workshop

# Optional customization
ocp4_workload_capacity_planning_workshop_deploy_rhacm: true
ocp4_workload_capacity_planning_workshop_namespace: my-workshop
```

## Testing Locally

Since you already have a running OpenShift cluster, you can deploy the workload directly:

```bash
# From agnosticd-v2 directory
ansible-playbook ansible/main.yml \
  -e @../agnosticd-v2-vars/my-vars.yml \
  -e ACTION=provision
```

## Removal

```bash
ansible-playbook ansible/main.yml \
  -e @../agnosticd-v2-vars/my-vars.yml \
  -e ACTION=destroy
```

## Author

Tosin Akinosho (takinosh@redhat.com)
