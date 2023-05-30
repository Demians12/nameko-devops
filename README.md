# Nameko Devops

This is a comprehensive solution to deploy a microservices-based application on an Amazon EKS cluster. The project is structured in three parts:

1. [Part 1](./part-1/README.md): Setting up an EKS Cluster using eksctl. 
2. [Part 2](./part-2/README.md): Preparing the Cluster for the Application.
3. [Part 3](./part-3/README.md): Deploying Epinio in EKS cluster and using crossplane to create new services to Epinio Catalog.

## Part 1: Setting up an EKS Cluster

In this phase, we focus on setting up a robust and scalable EKS cluster using `eksctl`. I walk through the steps to create a cluster, setup OIDC, create a nodegroup, and configure `kubectl`. Finally, verifying the cluster setup. Detailed instructions are available in the [Part 1 README](./part-1/README.md).

## Part 2: Preparing the Cluster for the Application

Once EKS cluster is up and running, It is necessary to prepare it for the application. This involves a series of steps including steps of deploying RabbitMQ, Postgres, and Redis on cluster and configure their secrets. Detailed instructions are available in the [Part 2 README](./part-2/README.md).

## Part 3: Deploying Application in EKS using Epinio

This part through the process of deploying [Epinio](https://docs.epinio.io/) in EKS. We'll use Crossplane with Epinio to improve our experience once these tools work great together. We'll create services in Epinio Catalog using Crossplane. Another way to create infrastructure is using eksctl-yaml. Instead of use a lot of commands we can make this process simple just using yaml to provision eks cluster.

## Conclusion

We understand that in the realm of cloud computing, things can quickly become overwhelming with a plethora of services, resources, and best practices. This project aims to approach the concept of developer experience. It also serves as a practical guide and a challenge 
