# PREPARE THE CLUSTER FOR THE APPLICATION

## INSTAL EBS CSI DRIVER TO THE DATABASE 

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
- Install
```bash
helm upgrade --install nginx ingress-nginx/ingress-nginx --namespace nginx --create-namespace --set controller.ingressClassResource.default=true
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
```bash
kubectl create secret generic cache-redis --from-literal=REDIS_PASSWORD=password_from_command
```
- Get POSTGRES secret: <br>
```bash
kubectl get secret --namespace nameko db-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d
``` 
```bash
kubectl create secret generic db-postgresql --from-literal=DB_PASSWORD=password_from_command
```
