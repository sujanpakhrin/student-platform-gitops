# Student Platform GitOps

This repository deploys a three-tier student records application: an Nginx frontend, a Flask API, and MySQL. Helm defines the Kubernetes resources. On Amazon EKS, Argo CD watches this repository and applies the selected Git revision.

## Start here

| What you want to do                         | Where to start                                                                                                                                                             |
| ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Run the application on your laptop          | [Application repository — Run locally](https://github.com/sujanpakhrin/student-platform-app#run-locally)                                                                   |
| Deploy the platform to a fresh AWS account  | [Deployment guide](docs/deployment-guide.md). Follow the **fresh installation** path in section 4 before enabling Argo CD.                                                 |
| Rebuild the lab using a retained MySQL disk | [Deployment guide — storage selection and recovery](docs/deployment-guide.md#4-choose-storage-before-enabling-gitops)                                                      |
| Review the capstone implementation          | [Final project report](docs/student-platform-final-report.pdf) · [Architecture diagram](docs/diagrams/architecture.svg) · [AWS network diagram](docs/diagrams/network.svg) |
| Diagnose a failed deployment                | [Troubleshooting guide](docs/troubleshooting-guide.md)                                                                                                                     |

## How the repositories fit together

| Repository                                                                         | Responsibility                                                                                                                                                                                                                  |
| ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [student-platform-app](https://github.com/sujanpakhrin/student-platform-app)       | Contains the frontend, backend, and database initialization SQL. GitHub Actions tests the application and publishes commit-tagged images to Docker Hub.                                                                         |
| [student-platform-infra](https://github.com/sujanpakhrin/student-platform-infra)   | Provisions the VPC, EKS cluster, private workers, IAM roles, and EBS CSI driver with Terraform.                                                                                                                                 |
| [student-platform-gitops](https://github.com/sujanpakhrin/student-platform-gitops) | Selects the application images and configures workloads, Services, storage, and backend autoscaling. The [Argo CD Application](argocd/student-platform.yaml) selects the repository, revision, chart path, and EKS values file. |

## Deploy to a fresh EKS cluster

The checked-in [EKS values](charts/student-platform/values-eks.yaml) reproduce the assessment’s two-volume recovery exercise. For a first deployment with an empty database, use the following sequence. The [deployment guide](docs/deployment-guide.md) contains the provisioning and installation commands.

1. **Verify the cluster.** Follow the infrastructure guide until the workers report `Ready` and the EBS CSI controller and node plugin have completed their rollouts.
2. **Select fresh storage in Git.** In your fork or deployment branch, remove the recovery-specific, top-level `database:` block from `charts/student-platform/values-eks.yaml`. Keep `ebsStorageClassName: student-platform-gp3`, the backend image tag, and the autoscaling settings. The chart inherits the database defaults from [values.yaml](charts/student-platform/values.yaml).
3. **Check the rendered claim** with the commands below. Commit and push the values change. Set `spec.source.repoURL` and `spec.source.targetRevision` in your Argo CD Application to the repository and branch containing that commit.
4. **Prepare the credentials.** Create the `student-platform` namespace and its `student-db-credentials` Secret with distinct `DB_PASSWORD` and `DB_ROOT_PASSWORD` values. Keep their values private. The [deployment guide](docs/deployment-guide.md#5-configure-the-secret-and-deploy-argo-cd) includes the commands.
5. **Deploy and verify.** Install Argo CD, apply your Application manifest, and enable Metrics Server for the CPU-based HPA. Confirm the PVC is `Bound`, all three Deployments are ready, and Argo CD reports `Synced` and `Healthy`.

From the GitOps repository root, check the storage selection before syncing:

```bash
helm lint ./charts/student-platform -f ./charts/student-platform/values-eks.yaml

helm template student-platform ./charts/student-platform \
  -f ./charts/student-platform/values-eks.yaml \
  --show-only templates/database.yaml \
  | grep -E 'storageClassName:|volumeName:|claimName:'
```

For a **fresh install**, expect `storageClassName: "student-platform-gp3"` and `claimName: "database-data"`, with no `volumeName` in the rendered claim. This allows the EBS CSI driver to provision a new disk for MySQL.

### Verify the application from your browser

The deployment guide creates the `capstone-eks` kubectl context. Use it explicitly if you also have Minikube configured.

```bash
kubectl --context=capstone-eks -n argocd get application student-platform
kubectl --context=capstone-eks -n student-platform get deploy,pvc,hpa
kubectl --context=capstone-eks -n student-platform port-forward svc/frontend 18083:80
```

Leave the port-forward running. Open <http://127.0.0.1:18083>, create two fictional students, open one by ID, and delete the other. Record the surviving ID. From a second terminal, run `curl -fsS http://127.0.0.1:18083/readyz` and retrieve `/api/students/<id>` using that ID. This exercises the request path from Nginx through Flask to MySQL.

## Storage used in the recovery exercise

During the recorded EKS rebuild, two claims were bound. The MySQL Deployment mounted `database-data-recovery-demo`.

| Claim                         | Role in the exercise                                                                                                                          |
| ----------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `database-data`               | Bound to the restored PV `student-mysql-restored`, preserving the original EBS disk. The original database credentials could not be verified. |
| `database-data-recovery-demo` | Mounted by MySQL to initialize a fresh database with the current Secret. This database held the student used in the backend recovery test.    |

To repeat this exercise, prepare both named resources before Argo CD syncs. For a populated disk, verify the volume ID, availability zone, and original database credentials. A `Bound` claim confirms the storage binding. A successful student lookup confirms the application can read the database.

## Images used in the verified EKS release

| Component      | Image tag                                      | Registry                                                                                     |
| -------------- | ---------------------------------------------- | -------------------------------------------------------------------------------------------- |
| Nginx frontend | `sha-4368a0d5f7701be833b04fdf62abcf7942c8c92f` | [Frontend on Docker Hub](https://hub.docker.com/r/codeykenny/student-platform-frontend/tags) |
| Flask backend  | `sha-c6ffbd980e9606f5a0590481145cbf502807c7b5` | [Backend on Docker Hub](https://hub.docker.com/r/codeykenny/student-platform-backend/tags)   |

The first CI release published both images with the same application commit tag. A later change to the EKS values promoted the backend independently. Argo CD synced GitOps commit `9fbfe33`, the ready backend pod showed the `sha-c6ff…` image, and `GET /version` returned `{"version":"v2"}`. Publishing another tag to Docker Hub leaves these pinned selections in effect until the GitOps configuration changes. [Release evidence](docs/screenshots/18-gitops-image-rollout.png)

## Verification evidence

### Helm deployment on Minikube

| Exercise                                                          | Observed result                                                                                                                        |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| [08 — Initial installation](docs/screenshots/08-helm-install.png) | Helm revision 1 deployed all three tiers. The pods became ready and the MySQL claim was bound.                                         |
| [09 — Student records](docs/screenshots/09-kubernetes-app.png)    | Through the port-forwarded frontend, I added two students, opened a record by ID, deleted Asha, and kept Nabin for the rollback check. |
| [10 — Helm upgrade](docs/screenshots/10-helm-upgrade.png)         | Revision 2 increased the backend from one ready replica to two.                                                                        |
| [11 — Helm rollback](docs/screenshots/11-helm-rollback.png)       | Revision 3 restored revision 1’s configuration. The backend returned to one ready replica, and I retrieved Nabin by ID.                |

<a id="day-5--eks-deployment-and-gitops-verification"></a>

### EKS deployment and GitOps verification

| Exercise                                                                         | Observed result                                                                                       |
| -------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| [16 — EKS database persistence](docs/screenshots/16-eks-storage.png)             | The MySQL claim was bound, and the saved student remained retrievable after database pod replacement. |
| [17 — Argo CD application status](docs/screenshots/17-argocd-synced-healthy.png) | Argo CD reported `Synced` and `Healthy` for the application on EKS.                                   |
| [18 — Git-driven release](docs/screenshots/18-gitops-image-rollout.png)          | The synced Git revision, running backend image, and `v2` response identified the deployed release.    |

### Monitoring and backend autoscaling

Metrics Server supplied CPU readings to the HPA. Each backend pod requested `50m` CPU. The HPA targeted 60% average CPU utilization and allowed two to four replicas. Prometheus and Grafana supplied the [monitoring dashboard](monitoring/student-platform-dashboard.json).

| Exercise                                                               | Observed result                                                                                                     |
| ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| [19 — Grafana dashboard](docs/screenshots/19-monitoring-dashboard.png) | The dashboard displayed cluster and workload health.                                                                |
| [20 — Before load](docs/screenshots/20-hpa-before.png)                 | The backend had two ready replicas, and the HPA reported 5% CPU utilization.                                        |
| [21 — Scale-up](docs/screenshots/21-hpa-scale-up.png)                  | A three-minute test against `/version` increased the backend to three available replicas.                           |
| [22 — Scale-down](docs/screenshots/22-hpa-scale-down.png)              | After load stopped, CPU fell to 4% while three replicas were still ready. The backend subsequently returned to two. |

### Scheduling and failure recovery

| Exercise                                                                     | Observed result                                                                                                                                                                |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [23 — Taints and tolerations](docs/screenshots/23-taint-toleration.png)      | Two pods targeted the same Minikube worker. The pod without a matching toleration stayed `Pending`. The pod with the toleration ran.                                           |
| [24 — Required node affinity](docs/screenshots/24-node-affinity.png)         | The scheduler placed the pod on the worker labelled `workload=student-api`.                                                                                                    |
| [25 — DaemonSet](docs/screenshots/25-daemonset.png)                          | One pod ran on each of the two Minikube nodes.                                                                                                                                 |
| [26 — Backend pod recovery](docs/screenshots/26-backend-pod-recovery.png)    | After I deleted one Flask pod on EKS, its ReplicaSet created a replacement. The backend returned to `2/2` ready, and the same student ID was retrievable through the frontend. |
| [27 — Argo CD drift recovery](docs/screenshots/27-argocd-drift-recovery.png) | I changed the live frontend replica count from one to two. Argo CD restored one replica and reported `Synced` and `Healthy` at the same Git revision, `4559a195`.              |
