# SETTING UP KARPENTER

## Set environment variables: <br>

```bash
export KARPENTER_VERSION=v0.27.5
```

## Then set the following environment variable:

```bash
export CLUSTER_NAME="lab1-eks"
```

```bash
export AWS_DEFAULT_REGION="us-east-1"
``` 
```bash
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
```
```bash
export TEMPOUT=$(mktemp)
```
# ----------------------------------------------------------------------
# CREATE A CLUSTER: <br>
- First create a key to import to aws and update the ssh keypath in the file: <br>

```bash
ssh-keygen -t rsa -b 2048 -f ~/.ssh/awskey
```

- Create a file controller-policy.json with the following permissions: <br>
```json
echo '{
    "Statement": [
        {
            "Action": [
                "ssm:GetParameter",
                "iam:PassRole",
                "ec2:DescribeImages",
                "ec2:RunInstances",
                "ec2:DescribeSubnets",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeLaunchTemplates",
                "ec2:DescribeInstances",
                "ec2:DescribeInstanceTypes",
                "ec2:DescribeInstanceTypeOfferings",
                "ec2:DescribeAvailabilityZones",
                "ec2:DeleteLaunchTemplate",
                "ec2:CreateTags",
                "ec2:CreateLaunchTemplate",
                "ec2:CreateFleet",
                "ec2:DescribeSpotPriceHistory",
                "pricing:GetProducts"
            ],
            "Effect": "Allow",
            "Resource": "*",
            "Sid": "Karpenter"
        },
        {
            "Action": "ec2:TerminateInstances",
            "Condition": {
                "StringLike": {
                    "ec2:ResourceTag/Name": "*karpenter*"
                }
            },
            "Effect": "Allow",
            "Resource": "*",
            "Sid": "ConditionalEC2Termination"
        }
    ],
    "Version": "2012-10-17"
}' > controller-policy.json
``` 

- Now create an policy using the file controller-policy.json: <br>
```bash
aws iam create-policy --policy-name KarpenterControllerPolicy-${CLUSTER_NAME} --policy-document file://controller-policy.json
```

- Next be ensure the yaml will get the values stored in the environment variables and create the cluster: <br>

```bash 
sed -e "s/\${CLUSTER_NAME}/${CLUSTER_NAME}/g" -e "s/\${AWS_ACCOUNT_ID}/${AWS_ACCOUNT_ID}/g" cluster.yaml | eksctl create cluster -f -
```

```yaml 
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ${CLUSTER_NAME}
  region: us-east-1
  version: '1.25'
  tags:
    karpenter.sh/discovery: ${CLUSTER_NAME}

iam:
  withOIDC: true
  serviceAccounts:
  - metadata:
      name: karpenter
      namespace: karpenter
    roleName: ${CLUSTER_NAME}-karpenter
    attachPolicyARNs:
    - arn:aws:iam::${AWS_ACCOUNT_ID}:policy/KarpenterControllerPolicy-${CLUSTER_NAME}
    roleOnly: true

managedNodeGroups:
  - name: managed-ng
    instanceType: t3.medium
    privateNetworking: true
    minSize: 2
    maxSize: 4
    volumeSize: 30
    desiredCapacity: 2
    ssh:
      allow: true
      publicKeyPath: ~/.ssh/awskey.pub
    iam:
      withAddonPolicies:
        externalDNS: true
        imageBuilder: true
        autoScaler: false # using karpenter
        certManager: true
        appMesh: true
        appMeshPreview: true
        awsLoadBalancerController: true
        xRay: true
        cloudWatch: true
        ebs: true
        fsx: true
        efs: true
```

- Run the commands: <br>
```bash
export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.endpoint" --output text)"
export KARPENTER_IAM_ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${CLUSTER_NAME}-karpenter"

echo $CLUSTER_ENDPOINT $KARPENTER_IAM_ROLE_ARN
```

```bash
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true
```
- If the role has already been successfully created, you will see:
`An error occurred (InvalidInput) when calling the CreateServiceLinkedRole operation: Service role name AWSServiceRoleForEC2Spot has been taken in this account, please try a different suffix.`

## # Logout of docker to perform an unauthenticated pull against the public ECR
```bash
docker logout public.ecr.aws
```

```bash
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter --version ${KARPENTER_VERSION} --namespace karpenter --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=${KARPENTER_IAM_ROLE_ARN} \
  --set settings.aws.clusterName=${CLUSTER_NAME} \
  --set settings.aws.defaultInstanceProfile=KarpenterNodeInstanceProfile-${CLUSTER_NAME} \
  --set settings.aws.interruptionQueueName=${CLUSTER_NAME} \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --wait
```
# --------------------------------------------------------------------------------------
# CREATE PROVISIONER

A single Karpenter provisioner is capable of handling many different pod shapes. Karpenter makes scheduling and provisioning decisions based on pod attributes such as labels and affinity. In other words, Karpenter eliminates the need to manage many different node groups.

Create a default provisioner using the command below. This provisioner uses securityGroupSelector and subnetSelector to discover resources used to launch nodes. We applied the tag karpenter.sh/discovery in the eksctl command above. Depending how these resources are shared between clusters, you may need to use different tagging schemes.

The ttlSecondsAfterEmpty value configures Karpenter to terminate empty nodes. This behavior can be disabled by leaving the value undefined.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]
  limits:
    resources:
      cpu: 1000
  providerRef:
    name: default
  ttlSecondsAfterEmpty: 30
---
apiVersion: karpenter.k8s.aws/v1alpha1
kind: AWSNodeTemplate
metadata:
  name: default
spec:
  subnetSelector:
    karpenter.sh/discovery: ${CLUSTER_NAME}
  securityGroupSelector:
    karpenter.sh/discovery: ${CLUSTER_NAME}
EOF
```
Karpenter is now active and ready to begin provisioning nodes.
# -----------------------------------------------------------------------------------
# EXAMPLE USE
- Scale up deployment

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
spec:
  replicas: 0
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      terminationGracePeriodSeconds: 0
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.7
          resources:
            requests:
              cpu: 1
EOF
````

```bash 
kubectl scale deployment inflate --replicas 5
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller
```
# -----------------------------------------------------------------------------------------
# SCALE DOWN DEPLOYMENT
Now, delete the deployment. After 30 seconds (ttlSecondsAfterEmpty), Karpenter should terminate the now empty nodes.

```bash
kubectl delete deployment inflate
```

```bash
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller
```
# ------------------------------------------------------------------------------------------

# ADD OPTIONAL MONITORING WITH GRAFANA

the following commands deploy a Prometheus and Grafana stack that is suitable for this guide but does not include persistent storage or other configurations that would be necessary for monitoring a production deployment of Karpenter. This deployment includes two Karpenter dashboards that are automatically onboarded to Grafana. They provide a variety of visualization examples on Karpenter metrics.

```bash
helm repo add grafana-charts https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
```bash
kubectl create namespace monitoring
```
```bash
curl -fsSL https://karpenter.sh/"${KARPENTER_VERSION}"/getting-started/getting-started-with-karpenter/prometheus-values.yaml | tee prometheus-values.yaml
helm install --namespace monitoring prometheus prometheus-community/prometheus --values prometheus-values.yaml
```
```bash
curl -fsSL https://karpenter.sh/"${KARPENTER_VERSION}"/getting-started/getting-started-with-karpenter/grafana-values.yaml | tee grafana-values.yaml
helm install --namespace monitoring grafana grafana-charts/grafana --values grafana-values.yaml
```

- The Grafana instance may be accessed using port forwarding.
```bash
kubectl port-forward --namespace monitoring svc/grafana 3000:80
```

- The new stack has only one user, admin, and the password is stored in a secret. The following command will retrieve the password:
```bash
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```
# ---------------------------------------------------------------------------------------

# CLEANUP

- Delete manually
```bash
kubectl delete node $NODE_NAME
```

- Delete the cluster
```bash
helm uninstall karpenter --namespace karpenter
aws cloudformation delete-stack --stack-name "Karpenter-${CLUSTER_NAME}"
aws ec2 describe-launch-templates --filters Name=tag:karpenter.k8s.aws/cluster,Values=${CLUSTER_NAME} |
    jq -r ".LaunchTemplates[].LaunchTemplateName" |
    xargs -I{} aws ec2 delete-launch-template --launch-template-name {}
eksctl delete cluster --name "${CLUSTER_NAME}"
```
