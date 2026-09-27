# TechPathway Capstone — Deployment Guide

Three-service Flask platform deployed to Amazon EKS via Amazon ECR.

## Services

| Service | Source | Port | Health | Datastore |
|---|---|---|---|---|
| main-app | weekly-call/ | 5111 | /health | SQLite (/app/app.db) |
| action-messages | action-messages/ | 5001 | /health | SQLite (/app/notifications.db) |
| techpathway-warehouse | techpathway-warehouse/ | 5002 | /health | SQLite (/app/warehouse.db) |

All three run at replicas=1. SQLite does not support concurrent writers.

## Prerequisites

- AWS CLI v2 configured (`aws sts get-caller-identity`)
- Docker Desktop running
- kubectl, eksctl, helm installed
- IAM permissions: ECR, EKS, EC2, CloudFormation, IAM

## Repository Layout

```
techpathway-capstone/
├── eks-cluster.yaml
├── k8s/
│   ├── main-app/{deployment,service,configmap}.yaml
│   ├── action-messages/{deployment,service,configmap}.yaml
│   ├── techpathway-warehouse/{deployment,service}.yaml
│   └── monitoring/prometheus-values.yaml
├── weekly-call/               # main-app source (unchanged)
├── action-messages/           # source (unchanged)
└── techpathway-warehouse/     # source (unchanged)
```

---

## Phase 1 — Clone

    git clone https://github.com/sholaolujobi/TechPathway--Website.git techpathway-capstone
    cd techpathway-capstone

---

## Phase 2 — ECR Repositories

    export AWS_REGION=us-east-1
    export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
    export ECR_REGISTRY=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

    aws ecr create-repository --repository-name main-app              --region $AWS_REGION
    aws ecr create-repository --repository-name action-messages       --region $AWS_REGION
    aws ecr create-repository --repository-name techpathway-warehouse --region $AWS_REGION

    aws ecr get-login-password --region $AWS_REGION | \
      docker login --username AWS --password-stdin $ECR_REGISTRY

If a repo already exists, delete it first with `--force` and re-create.

---

## Phase 3 — Build and Push

    docker build -t main-app ./weekly-call
    docker tag  main-app:latest $ECR_REGISTRY/main-app:latest
    docker push $ECR_REGISTRY/main-app:latest

    docker build -t action-messages ./action-messages
    docker tag  action-messages:latest $ECR_REGISTRY/action-messages:latest
    docker push $ECR_REGISTRY/action-messages:latest

    docker build -t techpathway-warehouse ./techpathway-warehouse
    docker tag  techpathway-warehouse:latest $ECR_REGISTRY/techpathway-warehouse:latest
    docker push $ECR_REGISTRY/techpathway-warehouse:latest

Verify:

    aws ecr describe-images --repository-name main-app              --region $AWS_REGION
    aws ecr describe-images --repository-name action-messages       --region $AWS_REGION
    aws ecr describe-images --repository-name techpathway-warehouse --region $AWS_REGION

---

## Phase 4 — EKS Cluster

Create `eks-cluster.yaml` at the repo root:

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: techpathway-cluster
  region: us-east-1
  version: "1.31"

managedNodeGroups:
  - name: techpathway-nodes
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 1
    maxSize: 3
    volumeSize: 30
    labels:
      role: worker

iam:
  withOIDC: true
```

Create the cluster:

    eksctl create cluster -f eks-cluster.yaml

Takes 15–20 minutes. Do not interrupt.

If the cluster already exists, eksctl returns "stack already exists" — that is not a failure. Skip to the verification step.

Verify:

    kubectl get nodes

Expected: 2 nodes in `Ready` state.

---

## Phase 5 — Apply Manifests

Substitute the ECR registry into all deployment YAMLs:

    sed -i "s|<ECR_REGISTRY>|$ECR_REGISTRY|g" k8s/*/deployment.yaml
    grep -rn "image:" k8s/

Confirm no `<ECR_REGISTRY>` placeholder remains. If the placeholder is already replaced from a previous run, `grep` and skip the `sed`.

Apply in dependency order:

    kubectl apply -f k8s/action-messages/
    kubectl apply -f k8s/techpathway-warehouse/
    kubectl create secret generic main-app-secrets \
      --from-literal=SECRET_KEY="$(openssl rand -hex 32)"
    kubectl apply -f k8s/main-app/

Verify:

    kubectl get pods
    kubectl get svc
    kubectl get cm

Expected: 3 pods in `Running` state, 1/1 READY.

If a pod is stuck in `ContainerCreating` or `Pending`, wait 60 seconds — the kubelet is still pulling images.

If a pod is in `CrashLoopBackOff`:

    kubectl logs <pod-name>
    kubectl describe pod <pod-name>

---

## Phase 6 — Validate

Port-forward all three services (one per terminal):

    kubectl port-forward svc/main-app              5111:5111
    kubectl port-forward svc/action-messages       5001:5001
    kubectl port-forward svc/techpathway-warehouse 5002:5002

Health checks:

    curl http://localhost:5111/health
    curl http://localhost:5001/health
    curl http://localhost:5002/health

All three must return `200`.

End-to-end test:

1. Open http://localhost:5111/store
2. Add a product to cart, checkout, place the order
3. Within ~3 seconds:
   - http://localhost:5002/ — the order appears in the warehouse `received` column
   - http://localhost:5001/ — a message with a tracking number appears, CC'd to m.olujobi1@gmail.com

---

## Phase 7 — Monitoring

    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update
    kubectl create namespace monitoring
    helm install monitoring prometheus-community/kube-prometheus-stack \
      --namespace monitoring --values k8s/monitoring/prometheus-values.yaml

    kubectl get pods -n monitoring
    kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80

Grafana: http://localhost:3000, admin / TechPathway2024!

### 7.1 — Verify the Prometheus data source

Open http://localhost:3000/connections/datasources

Confirm "Prometheus" is listed with a green "working" status.

If missing: Add data source → Prometheus → URL http://monitoring-kube-prometheus-prometheus:9090 → Save & test.

### 7.2 — Import community dashboards

kube-prometheus-stack does not install dashboards. Import three by ID.

Grafana → Dashboards → New → Import → paste ID → Load → select Prometheus → Import.

| ID | Dashboard |
|---|---|
| 15757 | Kubernetes / Views / Global |
| 15758 | Kubernetes / Views / Pods |
| 15759 | Kubernetes / Views / Namespaces |

### 7.3 — Verify dashboards show data

Open Kubernetes / Views / Pods (ID 15758).

Set Namespace = default, Pod = All. Confirm three rows appear:
main-app-*, action-messages-*, techpathway-warehouse-*

If a panel shows "No data":
- Set the time range to Last 15 minutes
- Confirm the data source is Prometheus, not `-- Grafana --`
- Open http://localhost:9090 → Status → Targets. All targets must be UP.

### 7.4 — Query your own apps in Explore

Grafana → Explore → data source = Prometheus. Paste each query:

    rate(container_cpu_usage_seconds_total{namespace="default", container!=""}[1m])

    container_memory_working_set_bytes{namespace="default", container!=""}

    sum(container_memory_working_set_bytes{namespace="default"}) by (pod)

Each returns live series for the three pods.

---

## Phase 8 — Cleanup

    kubectl delete -f k8s/main-app/
    kubectl delete -f k8s/action-messages/
    kubectl delete -f k8s/techpathway-warehouse/
    kubectl delete secret main-app-secrets

    helm uninstall monitoring -n monitoring
    kubectl delete namespace monitoring

    eksctl delete cluster --name techpathway-cluster --region us-east-1

    aws ecr delete-repository --repository-name main-app              --force --region us-east-1
    aws ecr delete-repository --repository-name action-messages       --force --region us-east-1
    aws ecr delete-repository --repository-name techpathway-warehouse --force --region us-east-1

Verify everything is gone:

    aws eks list-clusters --region us-east-1
    aws ecr describe-repositories --region us-east-1
    aws cloudformation list-stacks --region us-east-1 \
      --query "StackSummaries[?StackStatus!='DELETE_COMPLETE'].[StackName,StackStatus]" \
      --output table

Expected: empty clusters list, no ECR repos, no `eksctl-techpathway-cluster-*` stacks.

---

## Known Limitations

- SQLite is per-pod ephemeral. Pod restart = data loss. Acceptable for demo.
- All services at replicas=1. SQLite cannot handle concurrent writers.
- No ingress, no TLS. Access is via port-forward only.
- SMTP disabled by default. Emails are logged to the action-messages dashboard.
- No CI/CD. ECR push and `kubectl apply` are manual.

## Cost Reference

While the cluster is running:

- EKS control plane: $0.10/hour
- 2× t3.medium on-demand: ~$0.11/hour
- Total: ~$0.21/hour (~$1.68 overnight, ~$5 for 24 hours)
