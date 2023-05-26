# Nameko Devops

This is a comprehensive solution to deploy a microservices-based application on an Amazon EKS cluster. The project is structured in three parts:

1. [Part 1](./part-1/README.md): Setting up an EKS Cluster using eksctl. 
2. [Part 2](./part-2/README.md): Preparing the Cluster for the Application.
3. Part 3: Deploying the Application (Coming Soon).

## Part 1: Setting up an EKS Cluster

In this phase, we focus on setting up a robust and scalable EKS cluster using `eksctl`. We walk you through the steps to create a cluster, setup OIDC, create a nodegroup, and configure `kubectl`. Finally, we guide you on how to verify your cluster setup. Detailed instructions are available in the [Part 1 README](./part-1/README.md).

## Part 2: Preparing the Cluster for the Application

Once our EKS cluster is up and running, we need to prepare it for our application. This involves a series of steps including installing the EBS CSI driver for our databases, setting up Cert-Manager for handling certificates, and deploying controllers for traffic management. Additionally, we guide you on how to deploy RabbitMQ, Postgres, and Redis on your cluster and configure their secrets. Detailed instructions are available in the [Part 2 README](./part-2/README.md).

## Part 3: Deploying the Application (Coming Soon)

This part, which we will be adding soon, will guide you through the process of deploying your application. The application consists of three microservices: a Gateway, Products, and Orders. 

## Conclusion

We understand that in the realm of cloud computing, things can quickly become overwhelming with a plethora of services, resources, and best practices. This project aims to approach the concept of developer experience. It serves as a practical guide, helping devops engineers find solutions to minimize the overhead simplifying the developer's journey from code to cloud.
