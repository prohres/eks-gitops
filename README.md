# eks-gitops

GitOps configuration for the cluster built by [eks-infra](https://github.com/prohres/eks-infra).
An ArgoCD hub running outside the cluster installs add-ons and workloads from this repo.

```
ArgoCD hub ──watches──► this repo: bootstrap/ (ApplicationSets) ─► workloads/
    │  cluster Secret: label platform=eks + annotations eks-cluster-name / aws-region / vpc-id
    └──deploys──► EKS: aws-load-balancer-controller (Helm) ─► web (Ingress ─► internet-facing ALB)
```

## Layout

| Path | What |
|---|---|
| `bootstrap/` | ApplicationSets, applied once to the ArgoCD hub |
| `workloads/web/` | Demo app: nginx Deployment, Service, Ingress |

## Design decisions

- One ApplicationSet per component with a cluster generator (`platform=eks`): a cluster registered with
  that label gets everything, no per-cluster files.
- Environment data (cluster name, region, VPC id) lives in the cluster Secret on the hub, not in git.
- The Load Balancer Controller gets AWS permissions from Pod Identity (created in eks-infra for
  `kube-system/aws-load-balancer-controller`); the chart's service account name must match.
- The chart generates webhook TLS on every render (Helm `lookup` does not run in ArgoCD):
  `ignoreDifferences` + `RespectIgnoreDifferences` keep the first certificate.
- Chart and image versions are pinned; new releases wait about two weeks before adoption.

## Register a cluster

```bash
aws eks update-kubeconfig --region <region> --name <cluster-name> --alias <context>
argocd cluster add <context> --name <cluster-name> --label platform=eks \
  --annotation eks-cluster-name=<cluster-name> \
  --annotation aws-region=<region> \
  --annotation vpc-id=<vpc-id>     # eks-infra: terraform output -raw vpc_id
kubectl --context <hub-context> apply -f bootstrap/
```

## Remove

Order matters, otherwise the VPC destroy hangs on the ALB's ENIs and security groups:

1. `kubectl --context <hub-context> delete -f bootstrap/web.yaml`: the Ingress goes, the controller
   deletes the ALB. Wait until the ALB is gone.
2. `kubectl --context <hub-context> delete -f bootstrap/aws-load-balancer-controller.yaml`
3. `argocd cluster rm <context>`
4. `terraform destroy` in eks-infra.
