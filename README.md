# Student Platform GitOps

This repository contains the Helm chart and Argo CD configuration for the Student Management Platform. The same chart was tested on Minikube and deployed to Amazon EKS. Argo CD watches `main` and renders the chart with the EKS values.

## Project repositories

| Repository                                                               | What to find there                                                                                                 |
| ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| [Application](https://github.com/sujanpakhrin/student-platform-app)      | Nginx frontend, Flask API, MySQL initialization, Docker Compose, CI workflow, and published image evidence.        |
| [Infrastructure](https://github.com/sujanpakhrin/student-platform-infra) | Terraform configuration and verification for the VPC, EKS cluster, private workers, IAM roles, and EBS CSI driver. |
| [GitOps](https://github.com/sujanpakhrin/student-platform-gitops)        | Helm deployment, Argo CD release configuration, monitoring dashboard, and scheduling and recovery evidence.        |

## Deployment configuration

| Configuration                                                   | Purpose                                                                                                      |
| --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| [Helm chart](charts/student-platform/)                          | Deployments and Services for Nginx, Flask, and MySQL, plus application configuration and persistent storage. |
| [Default values](charts/student-platform/values.yaml)           | Base chart settings used for the Minikube deployment.                                                        |
| [EKS values](charts/student-platform/values-eks.yaml)           | EKS storage settings, backend image override, resource requests, and autoscaling policy.                     |
| [Argo CD Application](argocd/student-platform.yaml)             | Source repository, target revision, Helm chart path, and automated sync policy.                              |
| [Grafana dashboard](monitoring/student-platform-dashboard.json) | Dashboard configuration used for the monitoring demonstration.                                               |

The chart expects a Secret named `student-db-credentials` in the `student-platform` namespace with the keys `DB_PASSWORD` and `DB_ROOT_PASSWORD`. Create it before deploying the application. For an existing MySQL data volume, use credentials that match the accounts stored in that database.

To inspect the EKS manifests before deployment:

```bash
helm lint ./charts/student-platform -f ./charts/student-platform/values-eks.yaml
helm template student-platform ./charts/student-platform \
  -f ./charts/student-platform/values-eks.yaml
```

## Helm deployment on Minikube

The frontend and backend initially used the Docker Hub tag `sha-4368a0d5f7701be833b04fdf62abcf7942c8c92f`. The upgrade and rollback changed the backend replica count, making each revision visible in the running cluster.

| Evidence                                                       | Observed result                                                                                                                       |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| [08 — Helm installation](docs/screenshots/08-helm-install.png) | Revision 1 deployed all three tiers. The pods were ready and the MySQL claim was bound.                                               |
| [09 — Student records](docs/screenshots/09-kubernetes-app.png) | I added two students through the port-forwarded frontend, opened a record by ID, deleted Asha, and kept Nabin for the rollback check. |
| [10 — Helm upgrade](docs/screenshots/10-helm-upgrade.png)      | Revision 2 increased the backend from one ready replica to two.                                                                       |
| [11 — Helm rollback](docs/screenshots/11-helm-rollback.png)    | Revision 3 restored revision 1’s configuration. The backend returned to one ready replica, and I retrieved Nabin by ID.               |

## EKS deployment and GitOps verification

Argo CD deployed the chart from `main` using `values-eks.yaml`. The EKS configuration selects encrypted gp3 storage for MySQL and allows the backend image tag to change independently of the frontend.

| Evidence                                                                         | What to verify                                                                                                                                                    |
| -------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [16 — EKS database storage](docs/screenshots/16-eks-storage.png)                 | MySQL storage and persistence verification on EKS.                                                                                                                |
| [17 — Argo CD application status](docs/screenshots/17-argocd-synced-healthy.png) | Argo CD reported `Synced` and `Healthy`.                                                                                                                          |
| [18 — Git-driven backend release](docs/screenshots/18-gitops-image-rollout.png)  | Argo CD synced GitOps commit `9fbfe33`. The ready backend pod ran `sha-c6ffbd980e9606f5a0590481145cbf502807c7b5`, and `GET /version` returned `{"version":"v2"}`. |

### Storage used for the recovery demonstration

After the EKS rebuild, two PVCs existed in `student-platform`. Both were bound, but the MySQL Deployment mounted only the recovery-demo claim.

| PVC                           | Purpose                                                                                                                                                              |
| ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `database-data`               | Preserved the original EBS disk through the explicitly bound PV `student-mysql-restored`. The original database credentials could not be verified.                   |
| `database-data-recovery-demo` | A separately created claim used to initialize a fresh MySQL database with the current Secret. This database held the student used for the backend pod recovery test. |

When reproducing this recovery configuration, prepare the named restored PV and the separate demo claim before enabling Argo CD sync. The PVC’s `volumeName` preserves the original binding. The Deployment’s `claimName` selects the disk MySQL actually mounts.

## Monitoring and backend autoscaling

Metrics Server supplied CPU readings to the backend HPA. Prometheus and Grafana provided the monitoring dashboard. Each backend pod requested `50m` CPU, and the HPA targeted 60% average CPU utilization with a range of two to four replicas.

| Evidence                                                               | Observed result                                                                                              |
| ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| [19 — Grafana dashboard](docs/screenshots/19-monitoring-dashboard.png) | Dashboard used to observe cluster and application workload health.                                           |
| [20 — Before load](docs/screenshots/20-hpa-before.png)                 | The HPA reported 5% CPU utilization with two ready backend replicas.                                         |
| [21 — Scale-up](docs/screenshots/21-hpa-scale-up.png)                  | During a three-minute load test against `/version`, the backend reached three available replicas.            |
| [22 — Scale-down](docs/screenshots/22-hpa-scale-down.png)              | After load stopped, CPU fell to 4% while three replicas were still ready. The backend later returned to two. |

## Scheduling on Minikube

| Evidence                                                              | Observed result                                                                                                                   |
| --------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| [23 — Taint and toleration](docs/screenshots/23-taint-toleration.png) | Two pods targeted the same worker. The pod without the matching toleration stayed `Pending`; the tolerated pod ran on the worker. |
| [24 — Required node affinity](docs/screenshots/24-node-affinity.png)  | The scheduler placed the pod on the worker labelled `workload=student-api`.                                                       |
| [25 — DaemonSet](docs/screenshots/25-daemonset.png)                   | Two pods were ready, one on each node. The pod listing showed both node names.                                                    |

## Failure recovery on EKS

| Evidence                                                                     | Failure introduced                                                                | Recovery and verification                                                                                                            |
| ---------------------------------------------------------------------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| [26 — Backend pod recovery](docs/screenshots/26-backend-pod-recovery.png)    | I deleted one of two ready Flask pods.                                            | The ReplicaSet created a replacement. The backend returned to `2/2` ready, and I retrieved the same student ID through the frontend. |
| [27 — Argo CD drift recovery](docs/screenshots/27-argocd-drift-recovery.png) | I changed the live frontend replica count from one to two while Git declared one. | Argo CD restored one replica and reported `Synced` and `Healthy` at the same Git revision, `4559a195`.                               |
