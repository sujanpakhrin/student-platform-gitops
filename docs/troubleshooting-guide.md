# Student platform troubleshooting guide

Sujan Pakhrin · Recorded incidents from the capstone implementation.
See the [deployment guide](deployment-guide.md) for the normal installation sequence. In Fish, define the shortcut below before using the Kubernetes commands.

~~~fish
function keks
    kubectl --context=capstone-eks $argv
end
~~~

## Locate the failure

| Symptom | First observation | Question to answer |
|---|---|---|
| Terraform provisioning stopped | Read the failing resource and nested AWS error; verify caller and region. | Did credentials, authorization or account capacity block the action? |
| Argo CD shows OutOfSync | Inspect the Git revision, operation message and rendered EKS values. | Which field differs, and can Kubernetes accept that update? |
| Pod is Pending or ContainerCreating | Describe the pod and PVC; inspect events, storage class and node zones. | Has the scheduler selected a node, and has the disk attached? |
| Pod is Running with 0/1 ready | Read readiness events and container logs. | What did the application-level check return? |
| HPA target shows unknown | Run kubectl top pods and inspect CPU requests. | Is there a metrics sample and a denominator for CPU utilization? |

## 1. Terraform rejected the provider constraint or CIDR

**Observation.** Initialization rejected the string `-> 6.0`. A separate destroy run rejected a CIDR with an accidental leading greater-than sign.
**Resolution.** Use the valid provider constraint committed in providers.tf (the intended pessimistic constraint was `~> 6.0`). Supply the current admin IPv4 address followed by /32, without a leading symbol. Run terraform init and terraform validate before reviewing the plan. Successful validation clears the input error; AWS readiness is verified separately.

## 2. AWS CLI identity worked while Terraform could not assume the role

**Observation.** The CLI showed an assumed CapstoneTerraformDeploy role, but the AWS provider failed to load the profile chain.
**Resolution.** In this setup the capstone-dev profile represented the operator session, capstone-deploy assumed the deploy role, and capstone-tf exported those temporary credentials through a credential process. Verify locally:

~~~fish
aws configure set credential_process 'aws configure export-credentials --profile capstone-deploy --format process' --profile capstone-tf
aws sts get-caller-identity --profile capstone-tf --query Arn --output text
~~~

Use capstone-tf and us-west-2 consistently. The successful plan read the availability zones and displayed the intended network resources.

## 3. Worker provisioning failed or appeared stuck

**IAM denial.** CreateNodegroup reported missing iam:GetRole for AWSServiceRoleForAmazonEKSNodegroup. The existing policy covered the two project roles but omitted this service-linked role. Grant the named read permission on its correct ARN to the deployment role, then retry an updated plan. An AWS Console list permission is a separate concern from provisioning or Kubernetes access.

**Worker launch failure.** EKS stayed CREATING with an empty health issues list. Auto Scaling activity showed the chosen EC2 instance type was ineligible for the account's Free Tier restrictions. Select a supported type in the Terraform node-group configuration and review the replacement plan. Verify actual worker registration and Ready status with kubectl get nodes -o wide. Waiting alone does not fix an explicit launch denial.

## 4. EBS CSI add-on returned AccessDenied

**Observation.** The nested EKS error named eks:CreatePodIdentityAssociation on the student-platform cluster. Permission to describe an association did not grant permission to create it.
**Resolution.** Update the Terraform deployment role's scoped storage policy, then rerun the plan and apply. Keep the provisioning role distinct from the CSI controller's runtime IAM role. Verify the ebs-csi-controller Deployment, ebs-csi-node DaemonSet and ebs.csi.aws.com driver registration.

## 5. MySQL volume attached but login failed

**Observation.** The original 1 GiB EBS disk survived cluster teardown in us-west-2b. The rebuilt cluster needed a PV and PVC binding to reconnect it. Events showed SuccessfulAttachVolume. Early probes saw connection refused during startup; later probes consistently saw MySQL ERROR 1045 for student_app. A local socket login using the recreated root credential was also rejected.

~~~fish
keks -n student-platform describe pod -l app=database | sed -n '/Events:/,$p'
keks -n student-platform get pvc
keks get pv -o 'custom-columns=PV:.metadata.name,CLAIM:.spec.claimRef.name,DISK:.spec.csi.volumeHandle,RECLAIM:.spec.persistentVolumeReclaimPolicy'
~~~

**Resolution.** Recreating a Secret does not reset accounts already initialized on disk. We preserved the original PV with Retain, created database-data-recovery-demo, and selected it for the demo MySQL Deployment. Fresh initialization used the current Secret. Both claims became Bound to distinct volumes; MySQL was 1/1 and the backend 2/2. The pod recovery exercise used a newly created demo student. Historical records on the original disk remained unverified. Keep credentials out of screenshots.

## 6. Argo CD sync failed on an immutable PVC field

**Observation.** One EKS change omitted the storage class and attempted to clear student-platform-gp3. During a later rebuild, a different sync tried to clear the original claim's explicit volumeName, student-mysql-restored. Kubernetes rejected both attempted patches to bound claims.
**Resolution.** Compare the exact Git revision observed by Argo, the rendered chart with values-eks.yaml, and the live PVC. Restore the original storage class or volume binding in the chart, depending on which field the diff identifies. For the rebuild, the original PVC's volumeName preserves the retained PV; the Deployment's claimName selects the fresh demo PVC. These are different objects and different fields. Verify both rendered names before merging, then check the live deployment and PVCs.

~~~fish
keks -n argocd get application student-platform -o jsonpath='{.status.sync.revision}{"\n"}{.status.operationState.message}{"\n"}'
helm template student-platform ./charts/student-platform -f ./charts/student-platform/values-eks.yaml --show-only templates/database.yaml | rg 'storageClassName:|volumeName:|claimName:'
~~~

## 7. HPA absent or CPU unknown

**Missing object.** The chart included an HPA template, but the EKS values on the watched branch omitted required values. A separate indentation error placed requests outside resources. Inspect the rendered HPA and backend Deployment from the watched revision. Restore enabled=true, min=2, max=4, CPU target=60, and the backend request of 50m CPU. Lint alone did not expose the missing rendered request.

**Missing samples.** After rebuilding EKS, kubectl top returned “Metrics API not available” and the HPA displayed unknown/60%. Restore Metrics Server, wait for its APIService, and check actual readings. The backend subsequently showed 5%/60% and Argo CD was Synced and Healthy. The observed load test scaled from two replicas to three and returned to two after load ended. CPU dropped before the replica count fell.

## 8. Verify which controller restored service

**Deleted backend pod.** Record a known student and both pod names before deleting one named pod. The ReplicaSet creates a replacement. Verify the new name, 2/2 ready and retrieval of the same ID. Evidence: [26](screenshots/26-backend-pod-recovery.png).

**Configuration drift.** With self-healing enabled and Git declaring one frontend replica, a live patch temporarily requested two. Use JSONPath for patch output if kubectl rejects custom-columns. Observe the desired count returning to one, then compare Git revisions. The application ended Synced and Healthy at 4559a195. Evidence: [27](screenshots/27-argocd-drift-recovery.png).

## 9. Terraform state empty but AWS cleanup incomplete

**Observation.** Teardown encountered an EC2 DisassociateAddress denial. A separate read check failed because the deployment role lacked DescribeVolumes. A later inventory found the detached MySQL disk even after Terraform state was empty.
**Resolution.** Determine the failing action and the specific project resource. Use an appropriately authorized identity to check the exact EIP allocation and disk. When preserving data, record the EBS volume handle, zone, reclaim policy and credential source. Release or delete resources only after confirming ownership and the intended retention decision. Empty Terraform state confirms the workspace no longer tracks resources; AWS inventory supplies the independent cleanup evidence.
