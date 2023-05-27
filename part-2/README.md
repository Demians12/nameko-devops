# PREPARE THE CLUSTER FOR THE APPLICATION

## INSTAL EBS CSI DRIVER TO THE DATABASE 
That procedure must be made in the policies folder where are stored the policy files. <br>

1. create policy for ebs: 
```bash
aws iam create-policy --policy-name amazon_ebs_csi_driver --policy-document file://amazon_ebs_csi_driver.json
``` 

2. Get Worker node IAM Role ARN: <br>
```bash
kubectl -n kube-system describe configmap aws-auth
```

3. attach policy to a role. Here you must use your current node IAM role and ID account provided in the previous command: <br>
```bash
aws iam attach-role-policy --role-name eksctl-lab1-eks-nodegroup-lab1-ek-NodeInstanceRole-JFJFV2Y81725 --policy-arn arn:aws:iam::242364459859:policy/amazon_ebs_csi_driver
```

4. Deploy EBS CSI Driver:
```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"
```

5. Verify ebs-csi pods running: <br>
```bash
kubectl get pods -n kube-system
```

## INSTALL CERT-MANAGER <br>
```bash
helm repo add cert-manager https://charts.jetstack.io
helm repo update
helm install cert-manager --namespace cert-manager --create-namespace jetstack/cert-manager --set installCRDs=true --set extraArgs={--enable-certificate-owner-ref=true}
```

## INSTALL NGINX INGRESS CONTROLLER <br>
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

```bash
helm upgrade --install nginx ingress-nginx/ingress-nginx --namespace nginx --create-namespace --set controller.ingressClassResource.default=true
```

# OPTIONAL 
## INSTALL THE AWS LOAD BALANCER CONTROLLER
if you prefer to install the AWS LB controller instead Nginx controller you can follow these instructions:

- Download policy for controller. But the same policy file is stored in the policies folder in this repository: <br>
```bash
curl -o iam_policy_latest.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```
- Create IAM Policy using policy downloaded: <br>
```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy_latest.json
```
- Get worker node iam role: <br>
```bash
kubectl -n kube-system describe configmap aws-auth
```
- Create service account for controller:
```bash
eksctl create iamserviceaccount \
  --cluster=lab1-eks \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::242364459859:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```
- Add the eks-charts repository: <br>
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```
- List vpc-id. Use the vpc ID from the not default vpc: <br>
```bash
aws ec2 describe-vpcs
```
- Install the controller: <br>
```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=lab1-eks \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=vpc-0557114f896436a3d \
  --set image.repository=602401143452.dkr.ecr.us-east-1.amazonaws.com/amazon/aws-load-balancer-controller
```
- Verify that the controller is installed.
```bash
kubectl -n kube-system get deployment 
kubectl -n kube-system get deployment aws-load-balancer-controller
kubectl -n kube-system describe deployment aws-load-balancer-controller
```
- Verify Labels in Service and Selector Labels in Deployment
```bash
kubectl -n kube-system get svc aws-load-balancer-webhook-service -o yaml
kubectl -n kube-system get deployment aws-load-balancer-controller -o yaml
```
Observation:
1. Verify "spec.selector" label in "aws-load-balancer-webhook-service"
2. Compare it with "aws-load-balancer-controller" Deployment "spec.selector.matchLabels"
3. Both values should be same which traffic coming to "aws-load-balancer-webhook-service" on port 443 will be sent to port 9443 on "aws-load-balancer-controller" deployment related pods.

## CREATE INGRESS CLASS
### Navigate to root directory in part 2 to apply the ingress class
```bash
kubectl apply -f
```
- Verify ingress class
```bash
kubectl get ingressclass
```

## DEPLOY THE RABBITMQ, POSTGRES AND REDIS:

- Apply the namespace file to create the nameko namespace: <br>
```bash
kubectl apply -f namespace.yaml
```
- Install the rabbitmq: <br>
```bash
helm --namespace nameko install rabbitmq bitnami/rabbitmq
```
- Install the postgres: <br>
```bash
helm install db  bitnami/postgresql --set postgresqlDatabase=orders -n nameko
``` 
- Install redis cache: <br>
```bash
helm  install cache bitnami/redis -n nameko
``` 
### Storing sensitive information
- Get rabbitmq secret
```bash
kubectl get secret --namespace nameko rabbitmq -o jsonpath="{.data.rabbitmq-password}" | base64 -d
```
- Use the output of the previous command to store the secret in the etcd of the cluster using the following command: <br>
```bash
kubectl create secret generic rabbitmq --from-literal=RABBIT_PASSWORD=password_from_command
```
- Get Redis secret <br>
```bash
kubectl get secret --namespace nameko cache-redis -o jsonpath="{.data.redis-password}" | base64 -d
```
- Use the output of the previous command to store the secret in the etcd of the cluster using the following command:
```bash
kubectl create secret generic cache-redis --from-literal=REDIS_PASSWORD=password_from_command
```
- Get POSTGRES secret: <br>
```bash
kubectl get secret --namespace nameko db-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d
``` 
- Use the output of the previous command to store the secret in the etcd of the cluster using the following command:
```bash
kubectl create secret generic db-postgresql --from-literal=DB_PASSWORD=password_from_command
```
