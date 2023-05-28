# CHALLENGE
The first part of this project intends to create and prepare the cluster to host the application and Epinio.

## First Steps
### Creating the cluster 
First, we will create an EKS cluster in the AWS us-east-1 region. You can use the following command to create the cluster:

```bash
eksctl create cluster --name=lab1-eks \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --version="1.25" \
                      --without-nodegroup
```
## Check the created cluster with:
```bash 
eksctl get cluster
```

## 2. Key Pair Creation
```bash 
aws ec2 create-key-pair --key-name lab1-key >> ~/.ssh/lab1.pem
```
## 3. Setup OIDC Provider
**In order to enable IAM Roles for ServiceAccounts, we need to associate an IAM OpenID Connect (OIDC) provider with our cluster:** <br>

```bash 
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster lab1-eks \
    --approve
``` 
## 4. Nodegroup Creation
**We will create a nodegroup with t3.medium nodes in the private subnet:**
```bash
eksctl create nodegroup --cluster=lab1-eks \
                        --region=us-east-1 \
                        --name=lab1-eks-ng-private1 \
                        --node-type=t3.medium \
                        --nodes-min=2 \
                        --nodes-max=4 \
                        --node-volume-size=20 \
                        --ssh-access \
                        --ssh-public-key=lab1-key \
                        --managed \
                        --asg-access \
                        --external-dns-access \
                        --full-ecr-access \
                        --appmesh-access \
                        --alb-ingress-access \
                        --node-private-networking
```

## 5. Configure kubectl:
Configure kubectl to interact with your new cluster:

```bash
aws eks --region us-east-1 update-kubeconfig --name lab1-eks
```

## 6. Cluster Verification: <br>
- List cluster: <br>
`eksctl get cluster`

- List nodegroups in a cluster: <br>
`eksctl get nodegroup --cluster=lab1-eks`

- List nodes in current kubernetes cluster: <br>
`kubectl get nodes -o wide`

- kubectl context changed to new cluster: <br>
`kubectl config view --minify`

##  7. Verification of Roles, Security Groups and CloudFormation Stacks: <br>
Here are commands to help verify the roles, security groups, and CloudFormation stacks:

- Describe the instances to obtain the role name: <br>
`aws ec2 describe-instances --region us-east-1`

- Once you get the role name with the parameter `IamInstanceProfile` you can use to list the policies attached to the role: <br>
`aws iam list-role-policies --role-name <role-name>` <br>
This command will return a list of policy names attached to the role

- Get policy details: <br>
`aws iam get-role-policy --role-name <role-name> --policy-name <policy-name>`

- Describe the instances to obtain the role name: <br>
`aws ec2 describe-instances --region us-east-1`

- Get security group details: <br>
`aws ec2 describe-security-groups --group-ids <security-group-id> --region <region-name>`

- List Cloudformation stacks: <br>
`aws cloudformation describe-stacks --region <region-name>`

- Get details for the specific stack: <br>
`aws cloudformation describe-stacks --stack-name <stack-name> --region <region-name>`

- Get stack events: <br>
`aws cloudformation describe-stack-events --stack-name <stack-name> --region <region-name>`

## Login to Worker Node using Keypair: <br> 
- For Mac, Linux or Windows 10: <br>
`ssh -i lab-key.pem ec2-user@<Public-IP-of-Worker-Node>`


