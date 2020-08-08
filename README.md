

# Devops - Setting up a CI/CD Pipeline

## Overview

The document describes about setting up a CI/CD pipeline where the pipeline

* `source` the code from a Version Control System

* `Triggered` on any new commit in to the Version Control System

* `Builds`and install the `dependencies`

* `Creates` a docker image

* `Pushes` the docker image to a docker registry.

* `Implements` the deployment in a `blue-green` strategy 

* `Deploys` the app on a cloud managed, Kubernetes Orchestrated cluster

---

## Tool Stack

we will use

* `Jenkins` as a CI/CD tool, as it is a well known `stable` open source tool with huge community support and  a plethora of plugins.

* `AWS ec2` for hosting Jenkins master and slave. ec2 is a cloud based virtual machine provided by Amazon Web Services. It is secure, stable and provide a `public IP`. We need a public IP to access Jenkins from anywhere in the world.

* `AWS Elastic Container Registry` for storing the generated docker image. It is a fully-managed Docker container registry that makes it easy for developers to store, manage, and deploy Docker container images. You only pay for what you use / store.

* `AWS Elastic Kubernetes Service powered by Fargate` for deploying and managing our docker images. It is a fully managed Kubernetes service and with Fargate, it becomes serverless compute for containers. `You only pay for the amount of vCPU and memory resources that your pod needs to run`.

* `Github` as the public repository for source code management.

  

---

## CI CD Workflow

### High level Workflow Image

![CI CD Flow](https://dev-to-uploads.s3.amazonaws.com/i/6me8ebnomttrzjephhth.jpg)

### High Level Workflow

* The Developer pushes the code to github.

* Jenkins recogonizes this push or commit and initiates the build pipeline.

* The pipeline

  * installs the dependencies and build the `flask` application.

  * creates a docker image and `tags it based on the branch`.

  * pushes the docker image to an AWS ECR registry.

  * Applies `branch based` Kubernetes `Kustomization` and deploys to the `EKS cluster`.

    * Each `Kustomization`  points to a separate namespace depending on the branch.

    * We will use `Kubernetes namespaces` to achieve `blue-green deployments`.

      

## Infrastructure Architecture

### Architechture Diagram

![Infrastructure Architechture](https://dev-to-uploads.s3.amazonaws.com/i/vn866lrh0w5l25nqld6p.jpg)

Our deployment is managed and orchestrated by AWS EKS Fargate clusters.

### Components involved in EKS

* Ingress Controller
* Ingress Service
* AWS Elastic Load Balancer
* Service
* Deployment
* Pods

### Environment and Namespace

* An environment encompasses the necessary hardware(physical or virtual), Software dependencies, database, compiler/interpreter, network access for an application to run and is isolated from other environments unless explicitly connected.
* In our case, we use logical isolation using namespaces.
* We have three sets of all our components, excluding the ingress controller, each in its own namespace.
* We have 3 environments `production` or `blue`, `staging` or `green` and `develop`.
* The namespaces are also named accordingly such as
  *  `smallcase-blue` for blue / production environments.
  * `smallcase-green` for green / staging environments.
  * `smallcase-develop` for development environments.

### Network Traffic flow

* Each ingress will be assigned an AWS Elastic Load Balancer and it will be assigned an `IP` and `FQDN`.
* The user can hit the address in the browser.
* The ingress will communicate to a service in the same namespace.
* The service will internally load balance and communicate to the individual pods.
* Pods run our containers on the web app is hosted and hence the users are able to access the application.

## Branching Strategy

In our version control system, we will have three branches named

* master
* staging
* develop

#### The need of staging and develop branch.

Let us say that we only have master branch and our application is deployed based on the code in the master branch. We are releasing a new feature and the new feature has bug. Now, the customers who are using our application would be affected as they also face the same bug. But if we have develop and staging branch, before pushing the code to master, the code would be tested in develop and staging.

#### Link between the branches and deployment environments

The branch corresponds to its respective deployment environment and its namespace as follows

* master branch -> blue / production environment -> smallcase-blue namespace
* staging branch -> green / staging environment -> smallcase-green namespace
* develop branch -> develop environment -> smallcase-develop namespace

#### Branching rules

Below are the branching rules that has to be followed for easier management and not creating chaos. 

##### Master branch

- This is the main branch, whose code is deployed in the production/blue environment and being used by the customer in live.
- No new branch can be created from master.
- No commits can be directly made in master branch.
- It should accept pull requests only from the staging branch and it should not accept pull requests from other branches including the develop branch.
- Should not have unreleased code [ every commit in master must have a stable release associated ]
- The pull request from the staging branch should be reviewed and approved by the client/stakeholder/productOwner.(currently not applicable as I am the sole owner)

##### Staging branch

- The code in this branch will be deployed in staging environment
- No direct commits are allowed.
- It should accept pull request only from the develop branch and it should not accept pull requests from other feature or bugfix branches.
- It can make a pull request only to the master branch.
- No new branch can be created from staging. In case of any issue is found while testing or deploying the code in staging branch, devfix/bugfix branch should be created only from the develop branch.
- Should not have unreleased code [ every commit in staging must have a beta release associated ]
- The pull request from the develop branch should be reviewed and approved by the tech lead/scrum master.

##### Develop branch

- The code in this branch will be deployed in development environment.
- Direct commits are allowed.
- It can make a pull request only to the staging branch.

In a normal we will have more branches like feature and bug fix branches, but for now we will stick to these three branches.

As I am the only owner and I would not be able to approve myself if the rules are enforced, I have not applied them. When working in a team, these are some of the best practices that can be followed.

#### The deployment environments need to be similar

The deployment environments(infrastructure) needs to be as similar as possible because any bug would be found in development and user acceptance changes would be found in staging. Now, once everything is fixed, the staging(green) is as good as prod. So the push to production(blue), is as simple as a breeze, without any hiccups.

## Output

### Production / Blue Deployment

![Blue](https://dev-to-uploads.s3.amazonaws.com/i/6kjow6brf3qivkf6nfqh.JPG)

### Staging / Green Deployment

!![Green](https://dev-to-uploads.s3.amazonaws.com/i/3i1cwuhmvalsszeg4enf.JPG)

### Development Deployment

![Develop](https://dev-to-uploads.s3.amazonaws.com/i/u7a6e3gae19gtkewzyg8.JPG)

### Multi-Branch Pipelines

**Develop**

![Develop](https://dev-to-uploads.s3.amazonaws.com/i/d83j82ta4mjifw0udloh.JPG)

**Staging**

![Staging](https://dev-to-uploads.s3.amazonaws.com/i/297aj7xugz9frtcp2ggc.JPG)

**master**

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/i/e4itfupic6xqgsnxcfuw.JPG)
