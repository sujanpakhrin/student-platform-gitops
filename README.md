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
