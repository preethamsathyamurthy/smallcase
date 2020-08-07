

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



---

## Setting up Github as VCS Repository

### Clone the code into Github

Copy the URL from the provided Gitlab url repo and paste in the old repository section. Give the Repository Name smallcase. Now click Begin Import.

![Repository Clone](https://dev-to-uploads.s3.amazonaws.com/i/hmx5tsndrl4dzjnzp5wc.JPG)

![Cloned Code](https://dev-to-uploads.s3.amazonaws.com/i/ldmfmu0av5cutmv63e5a.JPG)



## Containerizing the Application

We assume that docker is installed in the machine we are analysing this. We know that we have a flask based application.

#### Requirements

We need

<ul>
    <li>Python 3</li>
    <li>nginx</li>
    <li>uWGSI</li>
</ul>

to host a flask based application

We can either choose to install these applications from a base ubuntu or alpine image or use an image which has python 3, nginx and uWGSI installed.

> As per the Docker best practices, to keep the image small, it is recommended to use an appropriate base image with necessary components installed, rather than starting with a generic ubuntu or alpine image. 

Hence we will use `tiangolo/uwsgi-nginx-flask:python3.8-alpine`. This image comes preinstalled with uwsgi, nginx and python3.8 with alpine as its base.

#### Checking the feasibility and structure of `tiangolo/uwsgi-nginx-flask:python3.8-alpine`

Let us test the image

```bash
#pull the image
docker pull tiangolo/uwsgi-nginx-flask:python3.8-alpine

#create a container
docker run -d -p 80:80 tiangolo/uwsgi-nginx-flask:python3.8-alpine

#check whether the container is running
docker container ls
CONTAINER ID        IMAGE                                         COMMAND                  CREATED             STATUS              PORTS                         NAMES
039541eab11f        tiangolo/uwsgi-nginx-flask:python3.8-alpine   "/entrypoint.sh /sta…"   0 minutes ago       Up 0 minutes        0.0.0.0:80->80/tcp, 443/tcp   quizzical_yalow
```

![init container](https://dev-to-uploads.s3.amazonaws.com/i/s17g93c3djjt1qnrmeh0.JPG)

We see that the app is up and running

Let us go inside the docker container using the container id or name and see the structure

```bash
docker exec -it 039541eab11f sh

#now we are inside the container

#check if nginx is running
if [ -e /var/run/nginx.pid ]; then echo "nginx is running"; fi
nginx is running

#check if python is installed and find the version
python --version
Python 3.8.2

#check uwsgi is installed and find the version
uwsgi --version
[uWSGI] getting INI configuration from /app/uwsgi.ini
2.0.18

#check the current directory
pwd
/app

#list the items in the directory
ls
__pycache__      main.py          prestart.sh      supervisord.pid  uwsgi.ini

#we see that the /app folder has the main flask files and also a uwsgi.ini file
cat main.py
import sys

from flask import Flask

app = Flask(__name__)


@app.route("/")
def hello():
    version = "{}.{}".format(sys.version_info.major, sys.version_info.minor)
    message = "Hello World from Flask in a uWSGI Nginx Docker container with Python {} (default)".format(
        version
    )
    return message


if __name__ == "__main__":
    app.run(host="0.0.0.0", debug=True, port=80)

#In our case, we have app.py
#Let's check uwsgi.ini
cat uwsgi.ini
cat uwsgi.ini
[uwsgi]
module = main
callable = app
```

From the above observation, we need to replace the /app folder contents with our application content and appropriate uwsgi file.

#### Using the tiangolo/uwsgi-nginx-flask:python3.8-alpine in our application

Clone the repository from github into local

```bash
#cloned file contents
ls
app.py  requirements.txt  templates
```

**create a uwsgi file**

```bash
vim uwsgi.ini
```

```ini
[uwsgi]
module = app
callable = app
master = true
```

Here module refers to app.py and callable refers to function app inside the app.py. The `master` option allows your application to keep running, so there is little downtime even when reloading the entire application.

```bash
ls
app.py  requirements.txt  templates  uwsgi.ini
```

**Creating a dockerFile**

Let us create a dockerfile which will use the requirement.txt file to install the dependencies we need and copy our application contents to the /app folder in the docker image.

```bash
vim Dockerfile
```

```dockerfile
FROM tiangolo/uwsgi-nginx-flask:python3.8-alpine

# copy over our requirements.txt file
COPY ./requirements.txt /tmp/

# upgrade pip and install required python packages
RUN pip install -U pip
RUN pip install -r /tmp/requirements.txt

# copy over our app code
#the requirements.txt file is copied twice
COPY  /app
```

**Build and Run**

**verifying if port 80 is available for use**

```bash
sudo nc localhost 80 < /dev/null; echo $?
1
```

if we get 1, it means the port is open.

We are building the application and running in port 80

```bash
docker build -t smallcase_flask .
docker run -p 80:80 -t smallcase_flask
```

![Docker Container Running](https://dev-to-uploads.s3.amazonaws.com/i/mhyp1a2uvx0f209hme35.JPG)

The application is up and running as a docker container.



---

## Creating an AWS Elastic Container Registry

### Assumption

It is assumed that the user has an AWS account with active subscription.

### Creating an AWS ECR

* Select Elastic Container Registry in the AWS services section.

* Click Create Repository

* give your repository a name and copy the name

  ![Create ECR Repo](https://dev-to-uploads.s3.amazonaws.com/i/mh6sz7plneoazzixzbfa.JPG)

Once Created, this will be your repository name. I have disabled Tag immutability as we want to use the tags blue, green and develop repeatedly.

### Prerequisites for pushing the image into ECR

* aws cli 2.0
* aws user with AmazonEC2ContainerRegistryFullAccess / Power User
* configure the aws profile

**installing aws cli 2.0**

In order to access aws, let us install awscli

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Verifying the installation

```bash
aws --version
aws-cli/2.0.35 Python/3.7.3 Linux/5.4.0-1018-aws botocore/2.0.0dev39
```

**Creating the ECS powerUser / ECS Full access role**

In the IAM services of AWS, create an user with AmazonEC2ContainerRegistryFullAccess / Power User.

**Configure the aws profile**

```bash
aws configure --profile smallcase
AWS Access Key ID [None]: *********
AWS Secret Access Key [None]: ************
Default region name [None]: us-east-2
Default output format [None]: json
```

**Get access token**

```bash
aws --profile smallcase ecr get-login-password

#output
eyJ****==
```

**Docker Login**

now login to the container registry using the access token obtained and the container url

```bash
docker login -u AWS -p eyJ****== https://374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app
```

**Push the image to the registry**

```bash
docker images
REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
smallcase_flask              latest              2c4103d1568f        2 hours ago         201MB
<none>                       <none>              0bc18b14f68a        2 hours ago         201MB
tiangolo/uwsgi-nginx-flask   python3.8-alpine    2d3ff13c3342        7 weeks ago         191MB
```

We are going to push `smallcase_flask`. Let us **give a tag to it with new repository URL and name**

```bash
docker tag smallcase_flask:latest 374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app:latest
```

Now let's push the image to the registry

```bash
docker push 374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app:latest
```

---

```bash
docker tag smallcase_flask:latest 374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app:blue
docker push 374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app:blue
```

![Registry Successful](https://dev-to-uploads.s3.amazonaws.com/i/mhzncidy38sdi1v3zzjj.JPG)

The image is successfully pushed to the repository.

---

### Creating an Amazon Elastic Kubernetes Service Cluster

#### Installing with `eksctl`

`eksctl` is a simple command line utility for creating and managing Kubernetes clusters on Amazon EKS. 

**Pre-Requisites**

* AWS CLI version 1.18.97 or later, 2.0.30 or later
* aws user with eks role
* Kubectl

```bash
aws --version
aws-cli/2.0.35 Python/3.7.3 Linux/5.4.0-1018-aws botocore/2.0.0dev39
```

We have 2.0.35 and we are good to proceed

We have already configured the aws profile.

#### installation

**install or upgrade eksctl**

```bash
#download and extract the latest release of eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

#Move the extracted binary to /usr/local/bin 
sudo mv /tmp/eksctl /usr/local/bin

#verify installation
eksctl version
0.24.0
```

Note: The `GitTag` version should be at least `0.24.0`. If not, check your terminal output for any installation or upgrade errors, or replace the address in step 1 with https://github.com/weaveworks/eksctl/releases/download/0.24.0/eksctl_Linux_amd64.tar.gz` and complete the above steps again

We have version 0.24.0 and we are good to proceed

**Install and configure kubectl**

Kubernetes uses the **kubectl** command-line utility for communicating with the cluster API server.

```bash
#Download the Amazon EKS-vended kubectl binary.
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.17.7/2020-07-08/bin/linux/amd64/kubectl

#Apply execute permissions to the binary
chmod +x ./kubectl

#move kubectl to a folder in our path
sudo mv ./kubectl /usr/local/bin

#########################################################
#incase we are upgrading kubectl or we have kubectl already installed for azure services,etc
	#it is recommended to create a $HOME/bin/kubectl folder, moving the binary to that folder, and ensuring that $HOME/bin comes first in your $PATH.
mkdir -p $HOME/bin && mv ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
	#If we do the previous step, optionally, Add the $HOME/bin path to your shell initialization file so that it is configured when you open a shell.
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bash_profile
##########################################################
#verify the installation
kubectl version --short --client
Client Version: v1.17.7-eks-bffbac
```

**connect to your aws profile**

```bash
aws configure
AWS Access Key ID [None]: *********
AWS Secret Access Key [None]: ************
Default region name [None]: us-east-2
Default output format [None]: json
```

#### create a Kubernetes cluster

I am creating a fargate powered Kubernetes-cluster named smallcaseCluster in us-east-2 region 

```bash
eksctl create cluster \
--name smallcaseCluster \
--version 1.17 \
--region us-east-2 \
--fargate \
--alb-ingress-access
```

It will take 15 to 20 minutes to create a Kubernetes cluster

**output**

```bash
[ℹ]  eksctl version 0.24.0
[ℹ]  using region us-east-2
[ℹ]  setting availability zones to [us-east-2c us-east-2b us-east-2a]
[ℹ]  subnets for us-east-2c - public:192.168.0.0/19 private:192.168.96.0/19
[ℹ]  subnets for us-east-2b - public:192.168.32.0/19 private:192.168.128.0/19
[ℹ]  subnets for us-east-2a - public:192.168.64.0/19 private:192.168.160.0/19
[ℹ]  using Kubernetes version 1.17
[ℹ]  creating EKS cluster "smallcaseCluster" in "us-east-2" region with Fargate profile
[ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=us-east-2 --cluster=smallcaseCluster'
[ℹ]  CloudWatch logging will not be enabled for cluster "smallcaseCluster" in "us-east-2"
[ℹ]  you can enable it with 'eksctl utils update-cluster-logging --region=us-east-2 --cluster=smallcaseCluster'
[ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "smallcaseCluster" in "us-east-2"
[ℹ]  2 sequential tasks: { create cluster control plane "smallcaseCluster", create fargate profiles }
[ℹ]  building cluster stack "eksctl-smallcaseCluster-cluster"
[ℹ]  deploying stack "eksctl-smallcaseCluster-cluster"

[ℹ]  creating Fargate profile "fp-default" on EKS cluster "smallcaseCluster"
[ℹ]  created Fargate profile "fp-default" on EKS cluster "smallcaseCluster"
[ℹ]  "coredns" is now schedulable onto Fargate
[ℹ]  "coredns" is now scheduled onto Fargate
[ℹ]  "coredns" pods are now scheduled onto Fargate
[ℹ]  waiting for the control plane availability...
[✔]  saved kubeconfig as "/root/.kube/config"
[ℹ]  no tasks
[✔]  all EKS cluster resources for "smallcaseCluster" have been created
[ℹ]  kubectl command should work with "/root/.kube/config", try 'kubectl get nodes'
[✔]  EKS cluster "smallcaseCluster" in "us-east-2" region is ready
```

Only when we see an output like `EKS cluster "smallcaseProduction" in "us-east-2" region is ready`, it means that the cluster is ready.

Verify the nodes are created and are ready.

```bash
kubectl get nodes
NAME                                                    STATUS   ROLES    AGE   VERSION
fargate-ip-192-168-152-201.us-east-2.compute.internal   Ready    <none>   24m   v1.17.6-eks-4e7f64
fargate-ip-192-168-189-227.us-east-2.compute.internal   Ready    <none>   24m   v1.17.6-eks-4e7f64
```

**Get the VPC**

```bash
eksctl get cluster --region us-east-2 --name smallcaseCluster -o yaml
- Arn: arn:aws:eks:us-east-2:374191519168:cluster/smallcaseCluster
  CertificateAuthority:
    Data: TmtJTjE0MmJ3UmVLdkhRZHN3RjB5VU14ST0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
  ...
  VpcId: vpc-0dbd63205b524a474
  RoleArn: arn:aws:iam::374191519168:role/eksctl-smallcaseCluster-cluster-ServiceRole-5WKVLN4U0U9W
  Status: ACTIVE
  Tags: {}
  Version: "1.17"
```

The VPC id is `vpc-0dbd63205b524a474`

##### Setup the AWS ALB ingress controller

 We will setup the  AWS ALB ingress controller so that we can allow external traffic to access our application. We will download the yaml file from Kubernetes-sigs

```bash
wget https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.8/docs/examples/rbac-role.yaml
wget https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v1.1.8/docs/examples/alb-ingress-controller.yaml
```

- The `rbac_role` manifest gives appropriate permissions to the ALB ingress controller to communicate with the EKS cluster we created earlier.
- The ALB ingress controller creates an Ingress Controller which uses ALB.

```bash
cat rbac-role.yaml
```

```yaml
cat rbac-role.yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/name: alb-ingress-controller
  name: alb-ingress-controller
rules:
  - apiGroups:
      - ""
      - extensions
    resources:
      - configmaps
      - endpoints
      - events
      - ingresses
      - ingresses/status
      - services
      - pods/status
    verbs:
      - create
      - get
      - list
      - update
      - watch
      - patch
  - apiGroups:
      - ""
      - extensions
    resources:
      - nodes
      - pods
      - secrets
      - services
      - namespaces
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/name: alb-ingress-controller
  name: alb-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: alb-ingress-controller
subjects:
  - kind: ServiceAccount
    name: alb-ingress-controller
    namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/name: alb-ingress-controller
  name: alb-ingress-controller
  namespace: kube-system
...
```

```bash
cat alb-ingress-controller.yaml
```

```yaml
# Application Load Balancer (ALB) Ingress Controller Deployment Manifest.
# This manifest details sensible defaults for deploying an ALB Ingress Controller.
# GitHub: https://github.com/kubernetes-sigs/aws-alb-ingress-controller
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: alb-ingress-controller
  name: alb-ingress-controller
  # Namespace the ALB Ingress Controller should run in. Does not impact which
  # namespaces it's able to resolve ingress resource for. For limiting ingress
  # namespace scope, see --watch-namespace.
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: alb-ingress-controller
  template:
    metadata:
      labels:
        app.kubernetes.io/name: alb-ingress-controller
    spec:
      containers:
        - name: alb-ingress-controller
          args:
            # Limit the namespace where this ALB Ingress Controller deployment will
            # resolve ingress resources. If left commented, all namespaces are used.
            # - --watch-namespace=your-k8s-namespace

            # Setting the ingress-class flag below ensures that only ingress resources with the
            # annotation kubernetes.io/ingress.class: "alb" are respected by the controller. You may
            # choose any class you'd like for this controller to respect.
            - --ingress-class=alb

            # REQUIRED
            # Name of your cluster. Used when naming resources created
            # by the ALB Ingress Controller, providing distinction between
            # clusters.
            # - --cluster-name=devCluster

            # AWS VPC ID this ingress controller will use to create AWS resources.
            # If unspecified, it will be discovered from ec2metadata.
            # - --aws-vpc-id=vpc-xxxxxx

            # AWS region this ingress controller will operate in.
            # If unspecified, it will be discovered from ec2metadata.
            # List of regions: http://docs.aws.amazon.com/general/latest/gr/rande.html#vpc_region
            # - --aws-region=us-west-1

            # Enables logging on all outbound requests sent to the AWS API.
            # If logging is desired, set to true.
            # - --aws-api-debug

            # Maximum number of times to retry the aws calls.
            # defaults to 10.
            # - --aws-max-retries=10
          env:
            # AWS key id for authenticating with the AWS API.
            # This is only here for examples. It's recommended you instead use
            # a project like kube2iam for granting access.
            # - name: AWS_ACCESS_KEY_ID
            #   value: KEYVALUE

            # AWS key secret for authenticating with the AWS API.
            # This is only here for examples. It's recommended you instead use
            # a project like kube2iam for granting access.
            # - name: AWS_SECRET_ACCESS_KEY
            #   value: SECRETVALUE
          # Repository location of the ALB Ingress Controller.
          image: docker.io/amazon/aws-alb-ingress-controller:v1.1.7
      serviceAccountName: alb-ingress-controller
...
```

Before we can apply these manifests, we need to uncomment and edit the following fields in the ALB Ingress Controller manifest above:

- `cluster-name`: The name of the cluster. In this case, we will use `fargate-tutorial-cluster`.
- `vpc-id`: VPC ID of the cluster. We saw how to access this field in the section above.
- `aws-region`: The region for your EKS cluster.
- `AWS_ACCESS_KEY_ID`: The AWS access key id that ALB controller can use to communicate with AWS. 
- `AWS_SECRET_ACCESS_KEY`: The AWS secret access key id that ALB controller can use to communicate with AWS. 

```bash
vim alb-ingress-controller.yaml
```

```yaml
# Application Load Balancer (ALB) Ingress Controller Deployment Manifest.
# This manifest details sensible defaults for deploying an ALB Ingress Controller.
# GitHub: https://github.com/kubernetes-sigs/aws-alb-ingress-controller
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: alb-ingress-controller
  name: alb-ingress-controller
  # Namespace the ALB Ingress Controller should run in. Does not impact which
  # namespaces it's able to resolve ingress resource for. For limiting ingress
  # namespace scope, see --watch-namespace.
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: alb-ingress-controller
  template:
    metadata:
      labels:
        app.kubernetes.io/name: alb-ingress-controller
    spec:
      containers:
        - name: alb-ingress-controller
          args:
            # Limit the namespace where this ALB Ingress Controller deployment will
            # resolve ingress resources. If left commented, all namespaces are used.
            # - --watch-namespace=your-k8s-namespace

            # Setting the ingress-class flag below ensures that only ingress resources with the
            # annotation kubernetes.io/ingress.class: "alb" are respected by the controller. You may
            # choose any class you'd like for this controller to respect.
            - --ingress-class=alb

            # REQUIRED
            # Name of your cluster. Used when naming resources created
            # by the ALB Ingress Controller, providing distinction between
            # clusters.
            - --cluster-name=smallcaseCluster

            # AWS VPC ID this ingress controller will use to create AWS resources.
            # If unspecified, it will be discovered from ec2metadata.
            - --aws-vpc-id=vpc-0dbd63205b524a474

            # AWS region this ingress controller will operate in.
            # If unspecified, it will be discovered from ec2metadata.
            # List of regions: http://docs.aws.amazon.com/general/latest/gr/rande.html#vpc_region
            - --aws-region=us-east-2

            # Enables logging on all outbound requests sent to the AWS API.
            # If logging is desired, set to true.
            # - --aws-api-debug

            # Maximum number of times to retry the aws calls.
            # defaults to 10.
            # - --aws-max-retries=10
          env:
            # AWS key id for authenticating with the AWS API.
            # This is only here for examples. It's recommended you instead use
            # a project like kube2iam for granting access.
            - name: AWS_ACCESS_KEY_ID
              value: AKIAVOH4NOXAHJAZNBE2

            # AWS key secret for authenticating with the AWS API.
            # This is only here for examples. It's recommended you instead use
            # a project like kube2iam for granting access.
            - name: AWS_SECRET_ACCESS_KEY
              value: rZ12tCZk3EcELT9UV4P6kTheDw74FW4R3qHpx33N
          # Repository location of the ALB Ingress Controller.
          image: docker.io/amazon/aws-alb-ingress-controller:v1.1.7
      serviceAccountName: alb-ingress-controller
```

##### Deploy the modified alb-ingress-controller

```
kubectl apply -f rbac-role.yaml

kubectl apply -f alb-ingress-controller.yaml
```

After applying the manifests, we can check the status of the ingress controller

```bash
kubectl get pods --all-namespaces
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   alb-ingress-controller-65b686fc99-rml8x   1/1     Running   0          98s
kube-system   coredns-56fbd5c8dd-j2zp4                  1/1     Running   0          53m
kube-system   coredns-56fbd5c8dd-rw5mb                  1/1     Running   0          53m
```

Now the elastic Kubernetes cluster is ready. Now we will be able to push images into the elastic cluster.

---

##### Creating Fargate Profiles for the namespaces in the kubernetes cluster

> EKS on Fargate by default only supports pods running in the `default` and `kube-system` namespaces.

```bash
eksctl create fargateprofile --namespace smallcase-green --cluster smallcaseCluster --region us-east-2

[ℹ]  creating Fargate profile "fp-5fea3d43" on EKS cluster "smallcaseCluster"
[ℹ]  created Fargate profile "fp-5fea3d43" on EKS cluster "smallcaseCluster"

eksctl create fargateprofile --namespace smallcase-blue --cluster smallcaseCluster --region us-east-2

eksctl create fargateprofile --namespace smallcase-develop --cluster smallcaseCluster --region us-east-2

```

Hence Created a fargate profile for the namespaces smallcase-green, smallcase-blue and smallcase-develop.

---



## Creating and Deploying the application to an EKS

* will create a Kubernetes namespace named smallCaseGreen
* will pull our image from ECR and create a deployment using the pulled image
* will create a service
* we will create an ingress service

**namespace.yaml**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: smallcase-green
```

Applying the above yaml will create a namespace smallcase-green.

**deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: smallcase-flask
  namespace: smallcase-green
  labels:
    app: smallcase-flask
    version: green
spec:
  replicas: 2
  selector:
    matchLabels:
      app: smallcase-flask
      version: green
  strategy: {}
  template:
    metadata:
      labels:
        app: smallcase-flask
        version: green
    spec:
      containers:
      - name: smallcase-flask
        image: 374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app:green
        ports:
          - containerPort: 80
        resources: {}
```

This on applying will create deployment smallcase-flask in the namespace smallcase-green and it controls pods with labels app : smallcase-flask and version: green. The pods use the image which we have created and pushed to our ecr with tag `green`. The application listens on port 80.

**service.yaml**

```yaml
kind: Service
apiVersion: v1
metadata:
  name: smallcase-flask
  namespace: smallcase-green
spec:
  selector:
    app: smallcase-flask
    version: green
  type: NodePort
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

This will create a service that listens to a deployments with label app: smallcase-flask and version: green. This will create a service of type nodeport. We can create a service of type loadbalancer which will create an aws alb.

**ingress.yaml**

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: smallcase-flask
  namespace: smallcase-green
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/scheme: internet-facing
spec:
  rules:
    - http:
        paths:
          - path: /
            backend:
              serviceName: smallcase-flask
              servicePort: 80
```

```bash
ls
deployment.yaml  ingress.yaml  namespace.yaml  service.yaml
```

### Apply the YAML files

Let's start with applying namespace

```bash
kubectl apply -f namespace.yaml
namespace/smallcase-green created

#validating
kubectl get namespace
NAME              STATUS   AGE
default           Active   3h14m
kube-node-lease   Active   3h14m
kube-public       Active   3h14m
kube-system       Active   3h14m
smallcase-green   Active   29s

```

Now let us create a service for our deployment

```bash
kubectl apply -f service.yaml
service/smallcase-flask created

#validating
kubectl get service -n smallcase-green
NAME              TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
smallcase-flask   NodePort   10.100.66.117   <none>        80:31075/TCP   20s
```

Now let us create our deployment

```bash
kubectl apply -f deployment.yaml
deployment.apps/smallcase-flask created

#validating
kubectl get pods -n smallcase-green
NAME                               READY   STATUS    RESTARTS   AGE
smallcase-flask-75b6bc9865-7qgjf   0/1     Pending   0          24s
smallcase-flask-75b6bc9865-t55zz   0/1     Pending   0          24s

#initially it will be in pending state and it will go into container creating state
kubectl get pods -n smallcase-green
NAME                               READY   STATUS              RESTARTS   AGE
smallcase-flask-75b6bc9865-82bm9   0/1     ContainerCreating   0          75s
smallcase-flask-75b6bc9865-f4tbk   0/1     Pending             0          75s

#voila its running
kubectl get pods -n smallcase-green
NAME                               READY   STATUS    RESTARTS   AGE
smallcase-flask-75b6bc9865-82bm9   1/1     Running   0          101s
smallcase-flask-75b6bc9865-f4tbk   1/1     Running   0          101s

#describing the pods and verifying
kubectl describe pods smallcase-flask-75b6bc9865-82bm9 -n smallcase-green
Name:                 smallcase-flask-75b6bc9865-82bm9
Namespace:            smallcase-green
Priority:             2000001000
Priority Class Name:  system-node-critical
Node:                 fargate-ip-192-168-166-224.us-east-2.compute.internal/192.168.166.224
Start Time:           Fri, 31 Jul 2020 13:07:25 +0000
Labels:               app=smallcase-flask
                      eks.amazonaws.com/fargate-profile=fp-5fea3d43
                      pod-template-hash=75b6bc9865
                      version=green
Annotations:          kubernetes.io/psp: eks.privileged
Status:               Running
IP:                   192.168.166.224
IPs:
  IP:           192.168.166.224
Controlled By:  ReplicaSet/smallcase-flask-75b6bc9865
Containers:
  smallcase-flask:
    Container ID:   containerd://85fc1456234ff9a9e39d2d77f1a9838aeba112df71a9b31defee4c8605ceef25
    Image:          374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app:green
    Image ID:       374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app@sha256:9abbc07010b347819747fd60b4beaddfd161487cb737f00746dade8369062198
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Fri, 31 Jul 2020 13:07:35 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-ktcvr (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-ktcvr:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-ktcvr
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From                                                            Message
  ----    ------     ----       ----                                                            -------
  Normal  Scheduled  <unknown>  fargate-scheduler                                               Successfully assigned smallcase-green/smallcase-flask-75b6bc9865-82bm9 to fargate-ip-192-168-166-224.us-east-2.compute.internal
  Normal  Pulling    55s        kubelet, fargate-ip-192-168-166-224.us-east-2.compute.internal  Pulling image "374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app:green"
  Normal  Pulled     49s        kubelet, fargate-ip-192-168-166-224.us-east-2.compute.internal  Successfully pulled image "374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app:green"
  Normal  Created    47s        kubelet, fargate-ip-192-168-166-224.us-east-2.compute.internal  Created container smallcase-flask
  Normal  Started    46s        kubelet, fargate-ip-192-168-166-224.us-east-2.compute.internal  Started container smallcase-flask

```

Now, let us create our ingress

```bash
kubectl apply -f ingress.yaml
ingress.extensions/smallcase-flask created
```

```bash
kubectl describe ingress -n smallcase-green smallcase-flask
```

```bash
Name:                 smallcase-flask-75b6bc9865-82bm9
Namespace:            smallcase-green
Priority:             2000001000
Priority Class Name:  system-node-critical
Node:                 fargate-ip-192-168-166-224.us-east-2.compute.internal/192.168.166.224
Start Time:           Fri, 31 Jul 2020 13:07:25 +0000
Labels:               app=smallcase-flask
                      eks.amazonaws.com/fargate-profile=fp-5fea3d43
                      pod-template-hash=75b6bc9865
                      version=green
Annotations:          kubernetes.io/psp: eks.privileged
Status:               Running
IP:                   192.168.166.224
IPs:
  IP:           192.168.166.224
Controlled By:  ReplicaSet/smallcase-flask-75b6bc9865
Containers:
  smallcase-flask:
    Container ID:   containerd://85fc1456234ff9a9e39d2d77f1a9838aeba112df71a9b31defee4c8605ceef25
    Image:          374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app:green
    Image ID:       374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app@sha256:9abbc07010b347819747fd60b4beaddfd161487cb737f00746dade8369062198
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Fri, 31 Jul 2020 13:07:35 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-ktcvr (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-ktcvr:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-ktcvr
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From                                                            Message
  ----    ------     ----       ----                                                            -------
  Normal  Scheduled  <unknown>  fargate-scheduler                                               Successfully assigned smallcase-green/smallcase-flask-75b6bc9865-82bm9 to fargate-ip-192-168-166-224.us-east-2.compute.internal
  Normal  Pulling    55s        kubelet, fargate-ip-192-168-166-224.us-east-2.compute.internal  Pulling image "374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app:green"
  Normal  Pulled     49s        kubelet, fargate-ip-192-168-166-224.us-east-2.compute.internal  Successfully pulled image "374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app:green"
  Normal  Created    47s        kubelet, fargate-ip-192-168-166-224.us-east-2.compute.internal  Created container smallcase-flask
  Normal  Started    46s        kubelet, fargate-ip-192-168-166-224.us-east-2.compute.internal  Started container smallcase-flask
root@ip-172-31-30-234:/home/ubuntu/smallcaseApp/kubernetes# kubectl apply -f ingress.yaml
ingress.extensions/smallcase-flask created
root@ip-172-31-30-234:/home/ubuntu/smallcaseApp/kubernetes# kubectl describe ingress -n smallcase-green smallcase-flask
Name:             smallcase-flask
Namespace:        smallcase-green
Address:          1dc416ec-smallcasegreen-sm-2779-688207968.us-east-2.elb.amazonaws.com
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host  Path  Backends
  ----  ----  --------
  *
        /   smallcase-flask:80 (192.168.123.85:80,192.168.166.224:80)
Annotations:
  alb.ingress.kubernetes.io/scheme:                  internet-facing
  alb.ingress.kubernetes.io/target-type:             ip
  kubectl.kubernetes.io/last-applied-configuration:  {"apiVersion":"extensions/v1beta1","kind":"Ingress","metadata":{"annotations":{"alb.ingress.kubernetes.io/scheme":"internet-facing","alb.ingress.kubernetes.io/target-type":"ip","kubernetes.io/ingress.class":"alb"},"name":"smallcase-flask","namespace":"smallcase-green"},"spec":{"rules":[{"http":{"paths":[{"backend":{"serviceName":"smallcase-flask","servicePort":80},"path":"/"}]}}]}}

  kubernetes.io/ingress.class:  alb
Events:
  Type    Reason  Age   From                    Message
  ----    ------  ----  ----                    -------
  Normal  CREATE  60s   alb-ingress-controller  LoadBalancer 1dc416ec-smallcasegreen-sm-2779 created, ARN: arn:aws:elasticloadbalancing:us-east-2:374191519168:loadbalancer/app/1dc416ec-smallcasegreen-sm-2779/902e843adabe5130
  Normal  CREATE  58s   alb-ingress-controller  rule 1 created with conditions [{    Field: "path-pattern",    PathPatternConfig: {      Values: ["/"]    }  }
```

Get the address field

> **dc416ec-smallcasegreen-sm-2779-688207968.us-east-2.elb.amazonaws.com**

It will take some time for the loadbalancer to load.

Let us open it in the browser and verify.

![green deployment](https://dev-to-uploads.s3.amazonaws.com/i/onbxuhkdmzs6qipscdok.JPG)

We can see that the application is up and running and is accessible in the browser. The application is running on a kubernetes managed docker container.

---

>  **Now we know the steps to install the dependencies, build a docker image, push to ECR registry and deploy to EKS.**



So let's get started with automating the process.



## Setting up the Jenkins Master and Slave

As mentioned previously, we are going to host the Jenkins server in an aws ec2 instance with ubuntu as the base os. 

#### Jenkins Master Installation

Connect to the ec2 instance via ssh

##### Prerequisites

* Java JRE has to be installed in the machine
* Port 8080 has to be open

**Java JRE installation**

```bash
sudo apt-get update
```

Let's install JRE from OPENJDK11

```bash
sudo apt install default-jre -y

#verification
java --version
openjdk 11.0.8 2020-07-14
OpenJDK Runtime Environment (build 11.0.8+10-post-Ubuntu-0ubuntu120.04)
OpenJDK 64-Bit Server VM (build 11.0.8+10-post-Ubuntu-0ubuntu120.04, mixed mode, sharing)
```

**Open Port 8080**

Port 8080 can be opened in the ec2 instance's security group's inbound rules.

##### Jenkins installation

The version of Jenkins included with the default Ubuntu packages is often behind the latest available version from the project itself. To ensure that we have the latest fixes and features, we use the project-maintained packages to install Jenkins.

Hence adding the project repository to the system

```
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
```

After the key is added the system will return with `OK`.

Next, we append the Jenkins Debian package repository address to the machine's `sources.list`:

```bash
sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
```

After both commands have been entered, we’ll run `update` so that `apt` will use the new repository.

Next we’ll install Jenkins and its dependencies.

```
sudo apt install jenkins -y
```

Now Jenkins is installed

**starting the Jenkins server**

```bash
sudo systemctl start jenkins
```

To see the status

```bash
sudo systemctl status jenkins

jenkins.service - LSB: Start Jenkins at boot time
     Loaded: loaded (/etc/init.d/jenkins; generated)
     Active: active (exited) since Sat 2020-08-01 07:18:11 UTC; 4min 14s ago
       Docs: man:systemd-sysv-generator(8)
    Process: 17090 ExecStart=/etc/init.d/jenkins start (code=exited, status=0/SUCCESS)
```

We see that jenkins is started successfully

Now, we will enable jenkins to run at startup

```bash
sudo systemctl enable jenkins
jenkins.service is not a native service, redirecting to systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable jenkins
```

**Assuming that Port 8080 has been opened as part of prerequisite activity**

Now when we try to open it in the browser using the machine's ip and port 8080, we see that jenkins has been succesfully installed.

![jenkins install](https://dev-to-uploads.s3.amazonaws.com/i/6tdmk29gkcuujzk2f4x9.JPG)

**now copy the password file from the path (specified in the UI) from the machine on which jenkin's is installed.**

You will be asked to install suggested plugins or select plugins to install.

Click install suggested plugins

Once installation is complete, we will be asked with a username and password for admin user.

I created an account with username: `smallcase` and password: `smallcase`

![jenkins successful](https://dev-to-uploads.s3.amazonaws.com/i/ld0oesr6y1yfqh6tzt6e.jpg)

Once Jenkins is configured successfully, we are able to see the jenkins home page.

For a production setup, it is good to configure a cloud based agent and we have many plugins using which we can use such as aws ec2(ec2 instances scale on demand) agents, Kubernetes orchestrated docker containers as agents, aws lambda objects as agents, etc.

*We will begin with a single permanent agent and we can shift to a cloud based agent when required.*

##### Create and establis ssh connectivity between the two ec2 instances

> we will set up a similar ec2 instance with ubuntu as the base OS. This ec2 instance will be our slave.

We will call the machine on which jenkins is hosted as master and the other machine as slave.

In Jenkins, the master and slave communicate via ssh. So we will create an ssh key in the master machine and share it with the slave machine, so that it can be used to connect to the slave machine.

In the master machine, create an ssh using the below command.

```bash
ssh-keygen
```

I gave an empty passphrase. Now SSH keygen is created.

Follow the same process in the slave machine

Now, coming back to the master machine.

```
cat ~/.ssh/id_rsa.pub
```

Copy the contents

Now, ssh into the slave machine once again and do the following.

```bash
cat >> ~/.ssh/authorized_keys
```

Paste the contents copied in the master machine and press ctrl+D

You can also edit that using a VIM editor.

> Now, basically, we have copied the public key of the master in the slave machine. Now, when master machine tries to connect to the slave machine, the slave machine will encrypt its credentials using the master machine's public key. The master machine decrypts the message using its private key and now it knows the credentials to log into the slave machine.

Now we can see whether its working by connecting to the slave machine from the master machine's terminal.

```bash
#ssh-copy-id username@ip -p 22
ssh ubuntu@172.31.31.172 -p 22

#bash profile changes
ubuntu@ip-172-31-31-172:~$
```

Now, the master machine can connect to the slave machine using ssh.

**Pre-requisites for Slave Machine**

* Java JRE has to be installed in the machine

Follow the steps which we used previously for installing java in master machine

###### Make the slave machine as jenkins agent

Now, go back to the Jenkins home page in the master machine and click create an agent.

You will see the below screen and give the node name as `jenkins-worker1` (any name of your choice) and select permanent agent. The machine which we are going to add will permanently be an agent for jenkins.

![Permanent agent](https://dev-to-uploads.s3.amazonaws.com/i/6qagwfkewja4f9zfafjv.JPG)

Click OK and you will se a page like below

![Agent Creation](https://dev-to-uploads.s3.amazonaws.com/i/uz3751wy1pgqndui6cej.JPG)

In the above page, the

* `# of executors`  - refer to the number of concurrent builds that can run in the VM.
* `remote root directory`  - this refers to the directory in the agent which will be used as the workspace for Jenkins executions
* `Launch method` - the method to connect to the agent. 
  * Select `Launch agents via SSH`
    * Give `HOST` as the private ip of the machine to be agent.
    * `Credentials` - click add button in the right
      * ![passkey](https://dev-to-uploads.s3.amazonaws.com/i/ofy1z6h5yhzyk0xpfrof.JPG)
      * The above window opens and select
        * `kind` as SSH Username with private key
        * `scope` as Global - this key will be accessible by any pipeline in this jenkins instance
        * `id` - give some name
        * `description` - give some description
        * `username` - Give ubuntu. This is the username which we would use to connect to the slave machine.
        * `Private Key` - Select Enter Directly and paste the private key of the master machine ( obtained by cat /home/ubuntu/.ssh/id_rsa )
        * `passphrase` - Enter the passphrase if given. We did not give any passphrase when creating the ssh key.
      * Click `Add`.
    * Now select the newly created key for `credentials`
    * `Host Key Verification strategy` - select known hosts file verification strategy ( would be added when we tried to connect to slave machine in the terminal via ssh)
  * `Availability` - Keep the agent as much possible
* Click save.

![Agent Online](https://dev-to-uploads.s3.amazonaws.com/i/29yw6fl46yx5wtzwei69.JPG)

![Agent Log Online](https://dev-to-uploads.s3.amazonaws.com/i/rixeto3j61pu41ef4c66.JPG)

On checking the log, we see that the agent has successfully connected and online.

###### Validating the agent connectivity 

We create a new freestyle project named agentCheck and in the build step, type hostname.

![Agent Check](https://dev-to-uploads.s3.amazonaws.com/i/oqs2i289arpa377to2r2.JPG)

*We see that the hostname belongs to the slave node and hence the agent works perfectly fine.*

###### Install Blue Ocean

**Blue Ocean** is a new user experience for **Jenkins** based on a personalizable, modern design that allows users to graphically create, visualize and diagnose Continuous Delivery (CD) Pipelines.

*click manage jenkins -> manage plugins -> select available in tabs*

*Search for Blue ocean -> select Blue Ocean -> click Download Now and Install after Restart*

Now Blue Ocean will be installed

Once jenkins restarts and we login, we can access blue ocean 

* by typing https://jenkinsURL/**blue**
* by clicking open blue ocean on the left side of the jenkins home page.

![open Blue OCean](https://dev-to-uploads.s3.amazonaws.com/i/a14boub4jevraki63dwi.JPG)

Once Blue Ocean opens, we will see the UI like below.

![Blue Ocean Home](https://dev-to-uploads.s3.amazonaws.com/i/9bjrp1opjpovqrnl0win.JPG)

##### Preparing the slave machine for performing our required CI/CD tasks

To be installed,

* Docker
* Python with its dependencies
* Python Virtual Environment

###### Installing Docker in the slave Machine

We know that we are going to build docker images. So let us install docker in the slave machine by connecting to it via ssh.

```bash
#update the repository
sudo apt-get update

#uninstall any old Docker software ( obviously not needed in our case but it is a good approach )
sudo apt-get remove docker docker-engine docker.io

#Install docker
sudo apt install docker.io -y

#Start docker 
sudo systemctl start docker

#Enable docker to start at run time
sudo systemctl enable docker

#verify docker installation
docker container ls
Got permission denied while trying to connect to the Dcoker Daemon socket at unix ...

#if such a error comes, we need to add our user to the dockers group
#create docker group
sudo groupadd docker

#Add your user to the docker group.
sudo usermod -aG docker ${USER}

Now we need to sign out and sign in to this session so that the group membership is reevaluated

#verifying docker without sudo
docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
0e03bdcc26d7: Pull complete
Digest: sha256:49a1c8800c94df04e9658809b006fd8a686cab8028d33cfba2cc049724254202
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

#another verification
docker container ls
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

Now docker is installed in the slave machine.

###### Install python 3 and its associated files in the slave machine

We know that we are building a flask application. So let us install python in our slave machine.

```bash
sudo apt-get install python3
python3 --version
	Python 3.8.2
```

install and verify pip and other python essentials in the slave machine

``````bash
sudo apt-get install python3-pip python3-dev build-essential libssl-dev libffi-dev python3-setuptools -y
pip3 --version
pip 20.0.2 from /usr/lib/python3/dist-packages/pip (python 3.8)
``````

###### Install python virtual environment

Let us set up a virtual environment in order to isolate our Flask application from the other Python files on the system and this causes isolation of python modules with other builds.

``` -ybash
apt-get install python3-venv -y
```

###### Configure Kubectl in slave machine to connect to our EKS

Configure your aws profile

```bash
aws configure
```

```bash
aws eks --region us-east-2 update-kubeconfig --name smallcaseCluster
us-east-2:374191519168:cluster/smallcaseCluster to /home/ubuntu/.kube/config

#verification
ubuntu@ip-172-31-30-234:~$ kubectl get nodes
NAME                                                    STATUS   ROLES    AGE   VERSION
fargate-ip-192-168-123-85.us-east-2.compute.internal    Ready    <none>   32h   v1.17.6-eks-4e7f64
fargate-ip-192-168-136-108.us-east-2.compute.internal   Ready    <none>   30h   v1.17.6-eks-4e7f64
fargate-ip-192-168-152-201.us-east-2.compute.internal   Ready    <none>   35h   v1.17.6-eks-4e7f64
fargate-ip-192-168-166-224.us-east-2.compute.internal   Ready    <none>   32h   v1.17.6-eks-4e7f64
fargate-ip-192-168-177-224.us-east-2.compute.internal   Ready    <none>   35h   v1.17.6-eks-4e7f64
fargate-ip-192-168-187-141.us-east-2.compute.internal   Ready    <none>   30h   v1.17.6-eks-4e7f64
fargate-ip-192-168-189-227.us-east-2.compute.internal   Ready    <none>   36h   v1.17.6-eks-4e7f64

```





---

## Establishing connectivity between Github and Jenkins Blue Ocean

Now, we will establish connectivity with Jenkins and Blue Ocean.

Click new pipeline in the Jenkins Blue Ocean.

choose Github. Select the link new access token, if you already don't have one.

![Blue Ocean 1](https://dev-to-uploads.s3.amazonaws.com/i/4hzazl5nrg41rb89g15u.JPG)

### Create a Person Access Token in Github

Lets create a personal access token in github. Give it a note for reference and click create.

![Blue Ocean PAT Creation](https://dev-to-uploads.s3.amazonaws.com/i/ahrvqn577hhudxaai8vz.JPG)

Copy the Personal Access Token

![PAT](https://dev-to-uploads.s3.amazonaws.com/i/112ws4d7w5v9lm0s8kj1.JPG)

### Blue Ocean Pipeline creation continuation

Now put the copied Personal access token blue ocean page and click connect. It will list our repositories.

![Repos Selection](https://dev-to-uploads.s3.amazonaws.com/i/avv21zffzi8rw1gokoqf.JPG)

We select the smallcase repository and click create pipeline.

![No Jenkins File](https://dev-to-uploads.s3.amazonaws.com/i/q1x1wmkig4xzbo8ked38.JPG)



We see that our branch has no JenkinsFile.

### Creating a WebHook in Github to Jenkins

We need a webhook to notify  Jenkins regarding any event like commit, pull-request.

In our project repository in github, click settings and then webhook. Click `Add Webhook`

In the payload URL type the `jenkinsURL/github-webhook/`. In our case, it is http://3.21.246.150:8080/github-webhook/.

Select Send me Everything on **events would you like to trigger this webhook?**

![Web Hook](https://dev-to-uploads.s3.amazonaws.com/i/r37ly417a4re845tgqg5.JPG)



Now the connection between blueOcean and Jenkins is established.



---

## Automation of tasks

### Adding a DockerFile

This dockerfile is the same as seen above

```dockerfile
FROM tiangolo/uwsgi-nginx-flask:python3.8-alpine

# copy over our requirements.txt file
COPY ./requirements.txt /tmp/

# upgrade pip and install required python packages
RUN pip install -U pip
RUN pip install -r /tmp/requirements.txt

# copy over our app code into the docker image
COPY . /app
```

### Adding the JenkinsFile

The Jenkins pipeline will use this JenkinsFile to perform the tasks. The pipeline is triggered on every commit on the connected github repository and this jenkins file creates a multibranch pipeline where tasks like initial checks, dependency installation and building is common, whereas creating a docker image, pushing it to ECR and deploying to EKS is branch specific.

```bash
pipeline {
  agent any
  stages {
    stage('initial check') {
      parallel {
        stage('check python version and pip3 version') {
          steps {
            sh '''python3 --version
                  pip3 --version'''
          }
        }
        
        stage('get current directory and file contents') {
          steps {
            sh '''pwd
                  ls
                  branch=$( git branch | grep \'^*\' |awk \'{print $2}\')
                  echo $branch'''
          }
        }
      }
    }

    stage('Installing dependencies and building the app') {
        steps {
            sh '''#!/bin/bash
                    pwd
                    #copy the build folders into a specific directory
                    mkdir build
                    cp -t ./build app.py templates requirements.txt uwsgi.ini -r
                    cd ./build
                    pwd
                    #install virutal env
                    python3 -m venv projectEnv
                    pwd
                    #login to virtualenv
                    source projectEnv/bin/activate
                    #to ensure that our packages will install even if they are missing wheel archives
                    pip install wheel
                    #installing dependencies
                    pip install -r requirements.txt
                    #moving out of virtual env
                    deactivate
                '''
        }
    }

    stage('perform unit tests if any') {
        steps {
            sh '''
                   pwd
                '''
        }
    }

    stage('remove the build folder') {
        steps {
            sh '''
                   rm -r build
                '''
        }
    }
        
    stage('build image for development') {
      when {
                branch 'develop' 
           }
      steps {
        sh 'docker build -t 374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app:develop .'
      }
    }

    stage('build image for staging') {
      when {
                branch 'staging' 
           }
      steps {
        sh 'docker build -t 374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app:green .'
      }
    }

    stage('build image for production') {
      when {
                branch 'master' 
           }
      steps {
        sh 'docker build -t 374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app:blue .'
      }
    }
    
    stage('Push image for development') {
        when {
                branch 'develop' 
        }
        steps {
            script {
                docker.withRegistry('https://374191519168.dkr.ecr.us-east-2.amazonaws.com', 'ecr:us-east-2:awsContainerCredential') {
                    sh "docker push 374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app:develop"
                }
            }
        }
    }
    
    stage('Push image for staging/green deployment') {
        when {
                branch 'staging' 
        }
        steps {
            script {
                docker.withRegistry('https://374191519168.dkr.ecr.us-east-2.amazonaws.com', 'ecr:us-east-2:awsContainerCredential') {
                    sh "docker push 374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app:green"
                }
            }
        }
    }

    stage('Push image for staging/blue deployment') {
        when {
                branch 'master' 
        }
        steps {
            script {
                docker.withRegistry('https://374191519168.dkr.ecr.us-east-2.amazonaws.com', 'ecr:us-east-2:awsContainerCredential') {
                    sh "docker push 374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app:blue"
                }
            }
        }
    }

    stage('Deploy Image in develop') {
      when {
                branch 'develop' 
        }
        steps {
        sh 'kubectl apply -f ./kubernetesConfigurations/develop/namespace.yaml'
        sh 'kubectl apply -k ./kubernetesConfigurations/develop'
        sh 'sleep 2m'
        sh 'kubectl rollout restart deployment smallcase-flask -n smallcase-develop'
        sh '''
            kubectl describe ingress -n smallcase-develop smallcase-flask | awk '/Address: /{print $2}'
           '''
      }
    }

    stage('Deploy Image in staging/ Green Deployment') {
      when {
                branch 'staging' 
        }
        steps {
        sh 'kubectl apply -f ./kubernetesConfigurations/staging/namespace.yaml'
        sh 'kubectl apply -k ./kubernetesConfigurations/staging'
        sh 'sleep 2m'
        sh 'kubectl rollout restart deployment smallcase-flask -n smallcase-green'
        sh '''
            kubectl describe ingress -n smallcase-green smallcase-flask | awk '/Address: /{print $2}'
           '''
      }
    }

    stage('Deploy Image in production/ Blue Deployment') {
      when {
                branch 'master' 
        }
        steps {
        sh 'kubectl apply -f ./kubernetesConfigurations/production/namespace.yaml'
        sh 'kubectl apply -k ./kubernetesConfigurations/production'
        sh 'sleep 2m'
        sh 'kubectl rollout restart deployment smallcase-flask -n smallcase-blue'
        sh '''
            kubectl describe ingress -n smallcase-blue smallcase-flask | awk '/Address: /{print $2}'
           '''
      }
    }
  }
}
```

### Installing plugins in Jenkins for pushing to ECR

* In manage jenkins -> manage plugins -> Available -> Select `CloudBees AWS Credentials Plugin` and click install.

* Once the plugin is installed, add a new aws credential by clicking manage jenkins -> manage aws credentials -> add new global credentials -> type `AWS Credentials`.

* ![AWS Credentials](https://dev-to-uploads.s3.amazonaws.com/i/cfhirzn0dcnsu55aj286.JPG)

  * Add the appropriate Access Key ID and Secret Access Key. 

  * In the above jenkinsFile

  * ```bash
    stage('Push image for staging/blue deployment') {
            when {
                    branch 'master' 
            }
            steps {
                script {
                    docker.withRegistry('https://374191519168.dkr.ecr.us-east-2.amazonaws.com', 'ecr:us-east-2:awsContainerCredential') {
                        sh "docker push 374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app:blue"
                    }
                }
            }
        }
    ```

    the first parameter of docker.withRegistry is the ECR repository uri, the second parameter is of the format `ecr:$region:$ourCredentialName`

  * `when` `branch` in the above jenkinsfile as the name suggests, will work execute when the branch is master.

### Kustomization.yaml

We are using Kustomization.yaml for reusing our yaml config as much as possible. The deployment,service and ingress Yaml files remains similar to what we have seen above.

Folder structure

![Kustomization Folder Structure](https://dev-to-uploads.s3.amazonaws.com/i/j0mktt15kaguh5jendq4.JPG)

Base Kustomization.yaml (./KubernetesConfigurations/yamlFiles/Kustomization.yaml)

``` yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: smallcase-develop
commonLabels:
  app: smallcase-flask
images:
  - name: 374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app # match images with this name
    newTag: develop # override the tag
    newName: 374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app # override the name
resources:
  - service.yaml
  - deployment.yaml
  - ingress.yaml
```

In the above, we are defining the namespace as common for all the resources, override the image tags, apply common labels.

Staging  Kustomization.yaml (./KubernetesConfigurations/staging/Kustomization.yaml)

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: smallcase-green
commonLabels:
  app: smallcase-flask
images:
  - name: 374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app # match images with this name
    newTag: green # override the tag
    newName: 374191519168.dkr.ecr.us-east-2.amazonaws.com/smallcase-app # override the name
bases:
- ../yamlFiles
```

This yaml file uses the above yaml file as base and applies/ overrides this configuration to the resources in base yaml.

>  This prevents us from creating multiple service deployment and ingresss yaml files , each for a deployment environment. We are only creating multiple kustomization.yaml s.

This might not seem that big of a use in this small example, but in a large production environment involving multiple deployment and service files, it becomes easier to manage with kustomization.yaml.

### Final Folder Structure

![Final Folder Structure](https://dev-to-uploads.s3.amazonaws.com/i/p9k6ko19wpmy7ivl7yco.JPG)

Now on pushing the code to github, we can see that the pipeline is automatically initiated in jenkins blueOcean.

![Blue Ocean Pick](https://dev-to-uploads.s3.amazonaws.com/i/em0mdc3hnav7t5832muv.JPG)

On clicking the build, we see the pipeline is executing and finally deploying the code to eks stack.

![master](https://dev-to-uploads.s3.amazonaws.com/i/5ixz41tpt0quzhggwtbi.JPG)

The Address of the EKS ingress/ELB is found in the last step of the branch's last stage.

![Address](https://dev-to-uploads.s3.amazonaws.com/i/lrnddta41l6fqrpyhf97.JPG)

Opening it we see the application is up and running.

![Application UP](https://dev-to-uploads.s3.amazonaws.com/i/t4ncvq2zmq6qseea58jy.JPG)



### Creating the additional Branches

Now let us create the develop and staging branches and make develop the default branch. We can also set rules like, preventing direct merge and allow only pull request. But I am the sole owner of this repo and I can't approve myself unless I ignore Administrator, which doesn't solve any purpose. But for any team, we have to do that.

![Branching](https://dev-to-uploads.s3.amazonaws.com/i/mr9rs3lq88lea2lgzfcr.JPG)

**I have made change in background colour to blue first in develop, then created a pull request to staging from develop and then another pull request to master from staging ** This is the correct approach to take. Similarly for staging, I updated the background colour to green in develop and created a pull request from develop into staging

The pipeline will be initiated. It will take some time around 5 minutes for the change to be reflected after deployment is done. We will also see the old black colour and the new colour intermittently, as it performs rolling updates(changes one by one in a pod). Once all the pods are updated, we will see only the new colour.

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



## Conclusion

Now we have CI/CD pipeline that satisfies our objectives mentioned in the overview. There are many more things that can be added like automated unit tests, automated semantic versioning and tagging, etc depending on the need. 
