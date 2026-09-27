# Tech Challenge 3 — EKS Containerized App with Autoscaling & Monitoring

Deploys a containerized nginx app to Amazon EKS via Helm, autoscales it with a Horizontal Pod Autoscaler (HPA), and monitors it with CloudWatch Container Insights.

## Repo structure

```
.
├── README.md
├── .gitignore
├── eks-app/
│   ├── index.html
│   ├── Dockerfile
│   └── eks-project/           # Helm chart (generated via `helm create`)
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
├── kubernetes/
│   └── hpa.yaml
└── monitoring/
    └── cwagentconfig.json     # only if you saved a local copy; otherwise omit — it's applied straight from the AWS repo URL
```

## Stack

- **Amazon EKS** — cluster provisioned via `eksctl`
- **Docker** — app image built and pushed to Docker Hub
- **Helm** — templated deployment of the app
- **metrics-server** — required for HPA to read CPU/memory usage (not bundled with EKS by default)
- **HPA** — CPU-based autoscaling, 1–5 replicas, 50% target
- **CloudWatch Container Insights** — monitoring via a DaemonSet + IRSA
- **Apache Bench (`ab`)** — load testing

## Setup

1. **Cluster**
   ```bash
   eksctl create cluster \
     --name enterprise-cluster \
     --region us-east-1 \
     --nodegroup-name standard-workers \
     --node-type t3.micro \
     --nodes 3 --nodes-min 3 --nodes-max 6 \
     --managed
   ```
   > Instance type must be Free Tier eligible on this account — only `t2.micro`/`t3.micro` qualify. `t2.small`/`t3.medium`/`t3.small` all fail with `InvalidParameterCombination` at the ASG launch step.

2. **Connect**
   ```bash
   aws eks --region us-east-1 update-kubeconfig --name enterprise-cluster
   ```

3. **Build & push the app image**
   ```bash
   cd eks-app
   docker build -t <your-dockerhub-username>/eks-app:1.0 .
   docker push <your-dockerhub-username>/eks-app:1.0
   ```

4. **Deploy via Helm**
   ```bash
   helm install eks-project ./eks-project
   ```

5. **Install metrics-server + patch for EKS's self-signed kubelet certs**
   ```bash
   kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
   kubectl patch deployment metrics-server -n kube-system --type='json' \
     -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'
   ```

6. **Apply the HPA**
   ```bash
   kubectl apply -f kubernetes/hpa.yaml
   ```

7. **CloudWatch monitoring (IRSA)**
   ```bash
   eksctl utils associate-iam-oidc-provider --region us-east-1 --cluster enterprise-cluster --approve

   eksctl create iamserviceaccount \
     --name cloudwatch-agent \
     --namespace amazon-cloudwatch \
     --cluster enterprise-cluster \
     --attach-policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy \
     --approve --override-existing-serviceaccounts

   kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cloudwatch-namespace.yaml
   kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cwagent/cwagent-serviceaccount.yaml

   curl -s "https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cwagent/cwagent-configmap.yaml" \
     | sed 's/{{cluster_name}}/enterprise-cluster/' | kubectl apply -f -

   kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/cwagent/cwagent-daemonset.yaml
   ```

## Load testing

`jstarcher/siege` had unresolvable DNS issues against both the ELB hostname and the raw ClusterIP on this cluster. `ab` (Apache Bench, via `httpd:2.4`) worked reliably instead:

```bash
kubectl run ab-test --image=httpd:2.4 --restart=Never -- ab -t 300 -c 200 "http://<cluster-ip>/"
```

## Notable issues hit & fixed during this build

| Issue | Fix |
|---|---|
| `t2.small`/`t3.medium` fail at node launch | Account restricted to Free Tier — use `t2.micro`/`t3.micro` |
| `metrics-server` shows `<unknown>` on HPA targets | Patch with `--kubelet-insecure-tls` |
| `jstarcher/siege` DNS resolution fails on raw IPs too | Switched to `ab` via `httpd:2.4` |
| `busybox wget` loops don't generate real concurrent load | Use `ab -c <concurrency>` instead |
| `iam-policy.json` 404s at the original doc's URL | AWS now uses the managed policy `arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy` — no custom policy file needed |
| `cwagentconfig` ConfigMap not found, pods stuck `ContainerCreating` | Repo moved this file under a `cwagent/` subdirectory; apply from the corrected path with `cluster_name` substituted via `sed` |

## Cleanup

```bash
kubectl delete svc eks-project
kubectl delete hpa eks-hpa --ignore-not-found
helm uninstall eks-project
eksctl delete iamserviceaccount --name cloudwatch-agent --namespace amazon-cloudwatch --cluster enterprise-cluster --wait
eksctl delete cluster --name enterprise-cluster --region us-east-1
```