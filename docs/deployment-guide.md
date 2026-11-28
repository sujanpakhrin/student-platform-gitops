# Student platform deployment guide

Sujan Pakhrin · Reference implementation: AWS us-west-2 · Commands use Fish on Linux.
Save this guide under docs/ in student-platform-gitops. Use [troubleshooting](troubleshooting-guide.md) for failed checkpoints.

## 1. Obtain the three repositories and identify the release

Install Git, Docker with Compose, Terraform, the AWS CLI, kubectl and Helm. The AWS profile must assume a role authorized for the project's EC2 network, EKS, IAM and add-on resources. The capstone used capstone-tf. Keep database credentials, Terraform state and saved plans private. Use the existing state/backend for an existing deployment.

~~~fish
mkdir -p ~/student-capstone
cd ~/student-capstone
# Run each clone once. Existing checkouts can be updated with git pull --ff-only on main.
git clone https://github.com/sujanpakhrin/student-platform-app.git
git clone https://github.com/sujanpakhrin/student-platform-infra.git
git clone https://github.com/sujanpakhrin/student-platform-gitops.git
for repo in student-platform-app student-platform-infra student-platform-gitops
    git -C $repo log -1 --format='%H %s'
end
~~~

| Image used for the demonstrated EKS v2 release | Registry |
|---|---|
| `codeykenny/student-platform-frontend:sha-4368a0d5f7701be833b04fdf62abcf7942c8c92f` | [Frontend tags](https://hub.docker.com/r/codeykenny/student-platform-frontend/tags) |
| `codeykenny/student-platform-backend:sha-c6ffbd980e9606f5a0590481145cbf502807c7b5` | [Backend tags](https://hub.docker.com/r/codeykenny/student-platform-backend/tags) |

The original CI run published both images under the first SHA. The later backend-only promotion selected the second SHA in the EKS values. Inspect the rendered manifests to see the images selected by your checkout.

## 2. Verify locally with Compose

In student-platform-app, copy .env.example to .env on first setup. Set distinct DB_PASSWORD and DB_ROOT_PASSWORD values. Keep the file private. Use the published frontend port configured for your checkout (8082 in the recorded demo).

~~~fish
docker compose config --quiet
and docker compose up -d --build --wait
and docker compose ps
and curl -fsS http://localhost:8082/readyz
~~~

Create two fictional students, retrieve one by ID and delete the other. To repeat the local persistence check, run docker compose down followed by docker compose up -d --wait, retaining the named volume. Fetch the same ID again.

## 3. Provision or verify AWS infrastructure

~~~fish
cd ~/student-capstone/student-platform-infra
set -gx AWS_PROFILE capstone-tf
set -gx AWS_DEFAULT_REGION us-west-2
aws sts get-caller-identity --query Arn --output text
git ls-files -- eks-storage.tf
terraform init
and terraform validate
and terraform plan -out=/tmp/student-platform-reviewed.tfplan
~~~

Supply your current public IPv4 in /32 notation for admin_ipv4_cidr. Inspect the plan for the VPC, subnets, routes, NAT, EKS cluster, workers, IAM roles, Pod Identity agent and EBS CSI driver. An empty-state rehearsal produced 26 additions. Existing installations can produce a different count. Apply once after review. If an apply is already running, let it finish.

~~~fish
terraform apply /tmp/student-platform-reviewed.tfplan
aws eks update-kubeconfig --name student-platform --alias capstone-eks --region us-west-2 --profile capstone-tf
function keks
    kubectl --context=capstone-eks $argv
end
keks get nodes -L topology.kubernetes.io/zone
keks -n kube-system rollout status deployment/ebs-csi-controller --timeout=5m
keks -n kube-system rollout status daemonset/ebs-csi-node --timeout=5m
keks get csidriver ebs.csi.aws.com
~~~

Checkpoint: ready workers, registered CSI driver and both CSI workloads ready.

## 4. Choose storage before enabling GitOps

Inspect both the live state and the EKS values. Editing a local file alone does not change the chart that Argo CD reads. Use the selected GitOps revision consistently.

~~~fish
cd ~/student-capstone/student-platform-gitops
keks get pvc -A
keks get pv
cat charts/student-platform/values-eks.yaml
~~~

**Fresh installation on an empty cluster.** Use a dedicated deployment branch or your own fork. In its EKS values, omit database.restoredVolumeName and database.existingClaim while keeping ebsStorageClassName: student-platform-gp3 and all other EKS settings. Commit those changes. Configure the Application's repoURL and targetRevision for that fork or branch. This lets the original chart-managed database-data PVC provision a fresh gp3 volume and the Deployment use it. Check the rendered claimName before sync.

**Repeat the recorded two-claim recovery configuration.** The recorded EKS values preserve student-mysql-restored and mount database-data-recovery-demo. Both names are prerequisites. For an existing cluster, verify its live bindings and reuse them. On a rebuilt cluster, inventory retained disks by their known CSI volume tags, region and zone. The original disk was identified by CSIVolumeName=pvc-c29a6ffc-0241-47db-aa61-58965d7902f2; the later demo disk by CSIVolumeName=pvc-53e7a10a-ed76-4281-a5f0-cd465bb8d459. These are historical identifiers. Check AWS for each current volume ID, state and zone.

~~~fish
aws ec2 describe-volumes --filters 'Name=tag:CSIVolumeName,Values=pvc-c29a6ffc-0241-47db-aa61-58965d7902f2,pvc-53e7a10a-ed76-4281-a5f0-cd465bb8d459' --query 'Volumes[].{Disk:VolumeId,Zone:AvailabilityZone,State:State,Size:Size}' --output table
~~~

For each retained disk you intend to use, confirm its ownership, size, filesystem and an eligible worker in its zone. Create the student-platform namespace first. Pre-bind one PV/PVC pair per disk using the following template, replacing the placeholders. For the preserved original use PV student-mysql-restored and PVC database-data. For the working demo disk use a separate PV name and PVC database-data-recovery-demo. Set the class to the same string used by the chart. Supply fsType if required for the disk's actual filesystem.

~~~yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: <pv-name>
spec:
  capacity: {storage: 1Gi}
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: student-platform-gp3
  claimRef: {namespace: student-platform, name: <claim-name>}
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: <verified-ebs-volume-id>
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values: [<verified-volume-zone>]
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {namespace: student-platform, name: <claim-name>}
spec:
  storageClassName: student-platform-gp3
  volumeName: <pv-name>
  accessModes: [ReadWriteOnce]
  resources: {requests: {storage: 1Gi}}
~~~

If the original disk is preserved but you want a new demo database, create the demo PVC only, omitting volumeName. It may stay Pending until a consumer is scheduled. If you pre-bind a retained demo disk, ensure the chart or Argo configuration that manages that claim preserves its intended binding. Database accounts on a populated disk require their original credentials. Binding a PVC does not establish that records are readable.

## 5. Configure the Secret and deploy Argo CD

Create the namespace if absent. Prepare a private local env file with DB_PASSWORD and DB_ROOT_PASSWORD; use the correct accounts for the selected disk. Enter its actual path at the prompt.

~~~fish
keks get namespace student-platform; or keks create namespace student-platform
read -P 'Path to private database env file: ' db_env
keks -n student-platform create secret generic student-db-credentials --from-env-file="$db_env"
~~~

On an existing install, verify the existing Secret instead of recreating it. Render the deployment and confirm the tag, class and claim selection. On a fresh EKS cluster install Argo CD; if already installed, verify its rollout. For a reproducible assessed run record the resolved Argo CD version or use a tested release tag in the manifest URL.

~~~fish
set chart ./charts/student-platform
helm lint $chart -f $chart/values-eks.yaml
and helm template student-platform $chart -f $chart/values-eks.yaml | rg 'image:|storageClassName:|claimName:|volumeName:'
keks get ns argocd; or keks create namespace argocd
keks -n argocd apply --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
and keks -n argocd rollout status statefulset/argocd-application-controller --timeout=5m
and keks -n argocd rollout status deployment/argocd-repo-server --timeout=5m
and keks apply -f argocd/student-platform.yaml
keks -n argocd get application student-platform
keks -n student-platform get deployment,pods,pvc
~~~

Allow time for sync, image pulls, EBS mounting and MySQL initialization. Expect database 1/1, frontend 1/1 and backend 2/2 at idle with EKS autoscaling enabled. If the HPA is degraded, complete the metrics step below.

## 6. Verify a request and the running images

~~~fish
keks -n student-platform get pods -l app=backend -o 'custom-columns=POD:.metadata.name,READY:.status.containerStatuses[0].ready,IMAGE:.spec.containers[0].image'
keks -n student-platform get pods -l app=frontend -o 'custom-columns=POD:.metadata.name,IMAGE:.spec.containers[0].image'
keks -n student-platform port-forward svc/frontend 18083:80
~~~

Leave the forward running. Open http://127.0.0.1:18083, create two records, retrieve one by its returned ID and delete the other. From a second terminal:

~~~fish
curl -fsS http://127.0.0.1:18083/api/students
read -P 'Surviving student ID: ' student_id
curl -fsS "http://127.0.0.1:18083/api/students/$student_id"
~~~

## 7. Restore monitoring on a fresh cluster

Install Metrics Server when absent. Verify the API and actual CPU readings before a load test.

~~~fish
keks apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
and keks -n kube-system rollout status deployment/metrics-server --timeout=3m
and keks wait --for=condition=Available apiservice/v1beta1.metrics.k8s.io --timeout=3m
keks -n student-platform top pods
keks -n student-platform get hpa backend
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack --kube-context capstone-eks --namespace monitoring --create-namespace --set alertmanager.enabled=false --set prometheus.prometheusSpec.retention=6h --wait --timeout 10m
keks -n monitoring port-forward svc/monitoring-grafana 3000:80
~~~

Open localhost:3000, sign in with the Grafana admin credentials from the monitoring-grafana Secret, import monitoring/student-platform-dashboard.json, and choose the Prometheus data source. Record the installed chart version for repeat runs. The baseline demonstration used 2-4 backend replicas, a 50m CPU request and a 60% target. Compare ready replicas and CPU before load, during a bounded test and after scale-down.

## 8. Demonstrate release and recovery

Promote a CI-published backend SHA tag through a reviewed GitOps values change. Compare the Application revision with the GitOps commit, the ready pod image tag with the app commit, and the version response with the release. For recovery, delete one named backend pod after recording both names and a known student ID. Wait for 2/2 ready and retrieve that ID again. To demonstrate drift, patch the live frontend from one to two replicas while Git declares one; observe self-healing restoring one at the same revision.

## 9. End the lab deliberately

Before teardown, decide which database and student records must survive. Record the PVC, PV, reclaim policy, actual CSI volume handle, zone and credential source privately. Back up required data and verify it before deleting any claims or cluster resources. Review a Terraform destroy plan separately. After cleanup, check project EBS disks and Elastic IP allocations directly in AWS. The separate unnamed VPC is outside this project.

### References

[Argo CD installation](https://argo-cd.readthedocs.io/en/stable/getting_started/) · [Metrics Server](https://github.com/kubernetes-sigs/metrics-server) · [Prometheus Community charts](https://prometheus-community.github.io/helm-charts/)
