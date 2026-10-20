# workbench-k8s-terraform

Owns the **shared network (VPC) + EKS cluster** for the **Workbench platform**
(Fase 4). The cluster is **shared by all microservices**, each isolated in its own
Kubernetes **namespace** (`os-service`, `billing-service`, `execution-service`,
`customer-service`). Cluster name: `workbench-platform-<env>` (`app_name=workbench-platform`).
One environment per branch:

| Branch | Environment | AWS account | State key |
|---|---|---|---|
| `develop` | dev | dev account | `k8s/develop/terraform.tfstate` |
| `main` | prod | prod account | `k8s/main/terraform.tfstate` |

## What it creates
- VPC (public + private subnets, single NAT by default) via `terraform-aws-modules/vpc`.
- EKS cluster + managed node group via `terraform-aws-modules/eks`, with `coredns`,
  `kube-proxy`, `vpc-cni` addons.
- `metrics-server` (Helm) — the cluster prerequisite for the services' HPA. Each HPA
  manifest lives in its own service repo (`k8s/hpa.yaml`).

## Consumed by (via `terraform_remote_state`)
- **workbench-database** → `private_subnet_ids`, `node_security_group_id`, `vpc_id`.
- **workbench-lambda** → `private_subnet_ids`, `node_security_group_id`.
- **microsserviços** (os/billing/execution/customer) → `cluster_name` para o
  `aws eks update-kubeconfig` no deploy de cada pipeline.

## Outputs
`vpc_id`, `private_subnet_ids`, `public_subnet_ids`, `node_security_group_id`,
`cluster_name`, `cluster_endpoint`, `cluster_ca_certificate` (sensitive), `region`.

## Configuration
Shared AWS auth/account/state-bucket come from **org-level secrets**, suffixed by
branch (`*_MAIN` / `*_DEVELOP`); `AWS_REGION` is a single org-level secret shared by
both branches (no suffix). See the app repo's `DEPENDENCIES.md` §5 for the full matrix.

## Local use
```bash
cp terraform.tfvars.example terraform.tfvars   # edit values
terraform init -backend=false                  # validation only (no remote state)
terraform validate
```
Real applies run in CI (`.github/workflows/cd.yml`) on pushes to `develop`/`main`.
