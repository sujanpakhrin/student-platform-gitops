## Helm deployment on Minikube

Deployed the student platform with Helm in the `student-platform` namespace.
The frontend and backend used the Docker Hub image tag
`sha-4368a0d5f7701be833b04fdf62abcf7942c8c92f`.

| Evidence                                                                 | Observed result                                                                                                                                      |
| ------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Initial installation](docs/screenshots/08-helm-install.png)             | Revision 1 deployed the three tiers. The pods were ready and the MySQL volume claim was bound.                                                       |
| [Student records in the browser](docs/screenshots/09-kubernetes-app.png) | I added two students and opened an individual record through the port-forwarded frontend. I then deleted Asha and kept Nabin for the rollback check. |
| [Helm upgrade](docs/screenshots/10-helm-upgrade.png)                     | Revision 2 increased the backend from one ready replica to two.                                                                                      |
| [Helm rollback](docs/screenshots/11-helm-rollback.png)                   | Revision 3 restored revision 1’s configuration. The backend returned to one ready replica, and I retrieved Nabin by ID.                              |

The chart expects a Secret named `student-db-credentials` in the release
namespace with keys `DB_PASSWORD` and `DB_ROOT_PASSWORD`.

## EKS deployment and GitOps verification

Argo CD deployed the student platform from the `main` branch of this repository. The Application renders `charts/student-platform` with `values-eks.yaml`. The EKS configuration selects encrypted gp3 storage for MySQL and sets the backend image tag independently of the frontend.

| Verification       | Result                                                                                                                                                                             | Evidence                                                           |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| Initial deployment | Argo CD reported `Synced` and `Healthy` for the application on EKS.                                                                                                                | [Argo CD status](docs/screenshots/17-argocd-synced-healthy.png)    |
| Git-driven release | Argo CD synced GitOps commit `9fbfe33`. The ready backend pod ran the image tagged `sha-c6ffbd980e9606f5a0590481145cbf502807c7b5`, and `GET /version` returned `{"version":"v2"}`. | [Backend v2 rollout](docs/screenshots/18-gitops-image-rollout.png) |

The [Argo CD Application](argocd/student-platform.yaml) defines the repository, chart path, and sync policy. The [EKS values](charts/student-platform/values-eks.yaml) identify the storage class and backend release.

## Day 6 — Monitoring and backend autoscaling

The backend HPA was configured for 2–4 replicas with a 60% CPU target and a
50m CPU request per pod. Before load, the HPA reported 5% utilization with
two ready replicas. During a three-minute test against `/version`, the backend
scaled to three available replicas. After the test completed, CPU fell to 4%
while three replicas were still ready. The backend later returned to two.

- [Grafana dashboard](docs/screenshots/19-monitoring-dashboard.png)
- [Before load](docs/screenshots/20-hpa-before.png)
- [Scale-up](docs/screenshots/21-hpa-scale-up.png)
- [Scale-down](docs/screenshots/22-hpa-scale-down.png)
- [Dashboard configuration](monitoring/student-platform-dashboard.json)
