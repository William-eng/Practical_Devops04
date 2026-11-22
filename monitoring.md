# Cohort 4 — Kubernetes Monitoring and Logging on Amazon EKS

> **Lab runbook | Windows PowerShell-first | Prometheus + Grafana + Loki + Grafana Alloy**
>
> This document records the successful log-collection configuration from the Cohort 4 lab and provides repeatable installation, validation, troubleshooting, and safe teardown guidance.

## 1. Learning objectives

- Understand the difference between **metrics** (Prometheus) and **logs** (Loki).
- Use **Grafana** to explore Kubernetes metrics and logs.
- Deploy **Grafana Alloy as a DaemonSet** to collect logs from each EKS worker node.
- Discover Pods dynamically, match `/var/log/pods/.../*.log`, and send entries to Loki.
- Verify the **data flow**, not just the health of Kubernetes Pods.
- Clean up the EKS lab without leaving billable AWS resources behind.

## 2. Architecture

```text
Amazon EKS cluster
├── Worker node A
│   ├── task-manager/backend Pod
│   ├── task-manager/frontend Pod
│   └── Alloy DaemonSet Pod ─┐
├── Worker node B           │
│   ├── other workloads     │
│   └── Alloy DaemonSet Pod ┤
│                           ▼
│                      Loki service ───► Grafana Explore (logs)
└── Kubernetes metrics ─► Prometheus ──► Grafana dashboards (metrics)
```

**Important:** The exact Prometheus/Grafana/Loki Helm chart versions and installation values used earlier in the lab were not supplied here. Commands below show how to inspect existing releases and an example fresh installation. Do not replace an existing deployment blindly.

## 3. Prerequisites

- An EKS cluster with at least one worker node.
- `aws` CLI, `kubectl`, `helm`, and suitable AWS IAM permissions.
- Working `kubectl` access to the cluster.
- Windows PowerShell 7 or Windows PowerShell 5.1 (commands use backticks for continuation).
- Network access to Helm repositories.

```powershell
aws sts get-caller-identity
kubectl config current-context
kubectl get nodes -o wide
helm version
kubectl get namespaces
```

> **Safety:** Check the AWS account, region, and Kubernetes context before installing or deleting anything.

## 4. Inspect the existing monitoring installation

```powershell
helm list -n monitoring
kubectl get pods -n monitoring -o wide
kubectl get svc -n monitoring
kubectl get daemonset -n monitoring
kubectl get pods -n task-manager -o wide
```

Typical components include Prometheus, Grafana, Loki, and Alloy. Names depend on your Helm release choices.

### Optional: example Helm installations for a _new_ cluster

These are reference commands, **not a claim about the exact commands used in the original lab**. Helm chart settings change between versions; pin tested versions for a production-ready training repository.

```powershell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -

# Prometheus + Grafana, for a new lab installation:
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack `
  --namespace monitoring

# Loki requires a values file appropriate for the chosen deployment mode,
# storage, and chart version. Inspect the chart before installing:
helm show values grafana/loki > .\monitoring\loki-reference-values.yaml

# Install Alloy using the complete values file in Section 5:
helm install alloy grafana/alloy `
  --namespace monitoring `
  --values .\monitoring\alloy.yaml
```

**Loki deployment note:** The lab uses the existing service `loki.monitoring.svc.cluster.local:3100`. The correct Loki chart configuration depends on whether it is deployed in SingleBinary or distributed mode, available storage, and chart version. Verify the existing installation with `helm list -n monitoring` and `kubectl get svc -n monitoring` before deploying Alloy.

## 5. Complete working `monitoring/alloy.yaml`

Create the file `monitoring/alloy.yaml` in your repository. **This is the verified configuration pattern that fixed the lab.**

```yaml
controller:
  type: daemonset

alloy:
  mounts:
    varlog: true

  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      memory: 256Mi

  configMap:
    content: |-
      discovery.kubernetes "pods" {
        role = "pod"
      }

      discovery.relabel "pod_logs" {
        targets = discovery.kubernetes.pods.targets

        rule {
          source_labels = ["__meta_kubernetes_pod_node_name"]
          action = "keep"
          regex = sys.env("K8S_NODE_NAME")
        }

        rule {
          source_labels = ["__meta_kubernetes_namespace"]
          target_label = "namespace"
        }

        rule {
          source_labels = ["__meta_kubernetes_pod_name"]
          target_label = "pod"
        }

        rule {
          source_labels = ["__meta_kubernetes_pod_container_name"]
          target_label = "container"
        }

        rule {
          source_labels = [
            "__meta_kubernetes_namespace",
            "__meta_kubernetes_pod_name",
            "__meta_kubernetes_pod_uid",
            "__meta_kubernetes_pod_container_name",
          ]
          separator = ";"
          regex = "([^;]+);([^;]+);([^;]+);([^;]+)"
          target_label = "__path__"
          replacement = "/var/log/pods/${1}_${2}_${3}/${4}/*.log"
        }
      }

      local.file_match "pod_logs" {
        path_targets = discovery.relabel.pod_logs.output
        sync_period = "10s"
      }

      loki.source.file "pod_logs" {
        targets = local.file_match.pod_logs.targets
        forward_to = [loki.write.default.receiver]
      }

      loki.write "default" {
        endpoint {
          url = "http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push"
        }
      }
```

### Why `local.file_match` matters

Kubernetes discovery produces a path containing a wildcard, for example `/var/log/pods/task-manager_backend-.../backend/*.log`. The file source in our failed test attempted to `stat` the literal `*.log` path and reported `no such file or directory`, even though `0.log` existed. The `local.file_match` component expands the wildcard to real file paths and passes the matches to `loki.source.file`.

The DaemonSet filters discovered Pods to its own node using `K8S_NODE_NAME`, avoiding duplicate reading of other nodes' files. The original lab already had this environment variable available in the Alloy container; confirm it in your own installation.

## 6. Deploy or update Alloy

```powershell
# Optional backup; choose a new name so an older backup is not overwritten.
Copy-Item .\monitoring\alloy.yaml .\monitoring\alloy-before-change.yaml

helm upgrade --install alloy grafana/alloy `
  --namespace monitoring `
  --values .\monitoring\alloy.yaml `
  --rollback-on-failure `
  --wait `
  --timeout 10m

kubectl rollout status daemonset/alloy -n monitoring
kubectl get pods -n monitoring -l app.kubernetes.io/name=alloy -o wide
```

Check that the active configuration has been loaded:

```powershell
kubectl get pods -n monitoring -l app.kubernetes.io/name=alloy -o wide
# Replace the pod name below if it changed.
kubectl exec -n monitoring alloy-mps6d -c alloy -- cat /etc/alloy/config.alloy
kubectl exec -n monitoring alloy-mps6d -c alloy -- printenv K8S_NODE_NAME
```

## 7. Verify Alloy metrics

Find the Alloy Pod running on the same node as the backend, then port-forward it. The pod name `alloy-mps6d` is the **historical name from this lab** and may change after a rollout.

**PowerShell terminal 1:**

```powershell
kubectl port-forward -n monitoring pod/alloy-mps6d 12346:12345
```

**PowerShell terminal 2:**

```powershell
$metrics = (Invoke-WebRequest http://localhost:12346/metrics).Content
$metrics -split "`n" |
  Select-String '^loki_source_file_files_active_total|^loki_write_sent_entries_total'
```

### Actual successful result (23 November 2026)

```text
loki_source_file_files_active_total{component_id="loki.source.file.pod_logs",component_path="/"} 30
loki_write_sent_entries_total{component_id="loki.write.default",component_path="/",host="loki.monitoring.svc.cluster.local:3100"} 16163
```

**Interpretation:** Alloy had 30 active log files and had sent 16,163 entries to the Loki endpoint. These are point-in-time values, not ongoing guarantees.

## 8. View logs in Loki and Grafana

Forward Loki if its service is named `loki`:

```powershell
kubectl port-forward -n monitoring svc/loki 3101:3100
```

From another terminal:

```powershell
Invoke-RestMethod "http://localhost:3101/loki/api/v1/labels?since=24h" |
  ConvertTo-Json -Depth 10
```

Find the actual Grafana service name:

```powershell
kubectl get svc -n monitoring | Select-String 'grafana'
```

If the service is named `grafana`:

```powershell
kubectl port-forward -n monitoring svc/grafana 3000:80
```

If installed through `kube-prometheus-stack`, the Grafana service name is often `kube-prometheus-stack-grafana`; use the actual name shown by `kubectl get svc`.

Open `http://localhost:3000`, choose **Explore → Loki**, and run:

```logql
{namespace="task-manager"}
```

```logql
{namespace="task-manager", container="backend"}
```

```logql
{namespace="task-manager"} |= "ERROR"
```

If Loki is not yet a Grafana data source, configure its URL using the Kubernetes service reachable from the Grafana Pod (typically `http://loki.monitoring.svc.cluster.local:3100` for this lab), rather than `localhost:3101`.

## 9. Troubleshooting: the issue we fixed

| Symptom                                      | Evidence                                            | Resolution                                                         |
| -------------------------------------------- | --------------------------------------------------- | ------------------------------------------------------------------ |
| Alloy Pods healthy but no logs               | `files_active_total = 0`, `sent_entries_total = 0`  | Inspect the actual file source and its metrics                     |
| File exists but cannot be tailed             | `head .../backend/0.log` succeeded                  | Basic file access was working                                      |
| File source tried to open wildcard literally | `stat .../backend/*.log: no such file or directory` | Insert `local.file_match` between discovery and `loki.source.file` |
| Helm deployed successfully but no log flow   | Healthy Kubernetes status alone                     | Check Alloy component metrics and Loki labels                      |
| Port-forward stale or wrong Pod              | Metrics from unexpected Pod                         | Recheck node placement and restart port-forward                    |

Useful diagnostics:

```powershell
kubectl logs -n monitoring alloy-mps6d -c alloy --since=15m |
  Select-String 'error|failed|tail|permission|file'

kubectl exec -n monitoring alloy-mps6d -c alloy -- `
  sh -c 'ls /var/log/pods/*task-manager*/*/*.log'

kubectl get pods -n task-manager -o wide
kubectl get pods -n monitoring -o wide
```

## 10. Safe EKS lab teardown — **do not run until ready**

Deleting the cluster is destructive. It removes the Kubernetes control plane and, depending on how the cluster was provisioned, can delete workloads and node groups. **Export anything you need first**, including screenshots, dashboards, manifests, credentials stored in Kubernetes Secrets, and application data. Database persistent volumes and AWS resources may require separate handling.

### A. Confirm account, region, cluster, and context

```powershell
aws sts get-caller-identity
aws configure get region
aws eks list-clusters --region <AWS_REGION>
kubectl config current-context
kubectl get nodes
```

Replace `<AWS_REGION>` and `<CLUSTER_NAME>` with verified values. **Never run these placeholders literally.**

### B. Export useful configuration before deletion

```powershell
New-Item -ItemType Directory -Force .\lab-backup | Out-Null
helm list -A | Out-File .\lab-backup\helm-releases.txt
kubectl get all -A -o wide | Out-File .\lab-backup\kubernetes-resources.txt
kubectl get pvc,pv -A | Out-File .\lab-backup\storage.txt
kubectl get ingress,svc -A | Out-File .\lab-backup\network-services.txt
```

The exported inventory is **not** a backup of database contents, PV data, or Grafana dashboards. Export those separately if needed.

### C. Check for external AWS resources

Before deleting EKS, inventory and handle resources created by Kubernetes controllers, including:

- `LoadBalancer` Services and Ingress resources backed by AWS ELB/ALB/NLB.
- EBS volumes and EBS CSI persistent volumes; inspect reclaim policies and snapshot data if needed.
- EFS file systems, if used.
- NAT gateways, Elastic IPs, VPC endpoints, and custom VPC infrastructure.
- ECR repositories/images, CloudWatch log groups, IAM roles/policies, and S3 buckets.
- RDS or other external databases, if created for the lab.

For a clean teardown, delete Kubernetes `Ingress` and `Service` resources managed by AWS load-balancer controllers **while the controllers and cluster are still running**, and wait for the associated AWS load balancers to disappear. Inspect all namespaces before deleting anything. Do not delete shared infrastructure used by other workloads.

### D. Identify the original provisioning method

Use the **same tool that created the cluster**:

**If created with `eksctl`:**

```powershell
eksctl get cluster --region <AWS_REGION>
eksctl delete cluster --name <CLUSTER_NAME> --region <AWS_REGION> --wait
```

**If created with Terraform:**

```powershell
# Run from the original Terraform project and workspace.
terraform workspace show
terraform state list
terraform plan -destroy
# Review the full plan and only then:
terraform destroy
```

**If created through CloudFormation or the AWS Console:** follow the original stack/resource dependency order. Avoid mixing deletion methods, which can leave orphaned resources or broken infrastructure state.

> **Do not run `eksctl delete cluster` for a Terraform-managed cluster.** That can bypass Terraform state and complicate cleanup.

### E. Confirm billing cleanup

After deletion, check the AWS Console for remaining EKS clusters, EC2 instances, load balancers, EBS volumes/snapshots, NAT gateways, Elastic IPs, RDS instances, and CloudWatch logs in the **correct region and account**. Inspect AWS Billing / Cost Explorer later as cost data can lag.

## 11. What students should remember

1. **Prometheus** collects metrics; **Loki** stores and queries logs; **Grafana** visualizes both; **Alloy** collects and forwards telemetry.
2. A healthy Pod does not prove that telemetry is flowing.
3. A DaemonSet runs a collector on each eligible worker node.
4. `discovery.kubernetes` discovers Pods; `discovery.relabel` attaches useful labels and builds paths.
5. `local.file_match` expands wildcard file paths into real files; `loki.source.file` reads them; `loki.write` sends entries to Loki.
6. Check actual metrics and errors to troubleshoot methodically.
7. Always remove temporary cloud lab resources after exporting what you need.

---

**Lab outcome:** Alloy DaemonSet healthy on two worker nodes; 30 active log files; 16,163 log entries sent to Loki. Grafana Explore verification is a separate final check.
