
# Setting Up a Kubernetes Cluster on AWS with Terraform and Ansible  

This guide provides a detailed walkthrough for provisioning a Kubernetes cluster on AWS using Terraform and Ansible

## Goals:  

- AWS VPC
- 1 EC2 instances for HA Kubernetes Control Plane: Kubernetes API, Scheduler and Controller Manager
- 1 EC2 instances for *etcd* cluster
- 1 EC2 instances as Kubernetes Workers (aka Minions or Nodes)
- Kubenet Pod networking (using CNI)
- HTTPS between components and control API
- Sample *nginx* service deployed to check everything works

*This is a learning tool, not a production-ready setup.*  

## Pre-requisites  

Before starting, ensure you have the following tools installed on your control machine:

Install requirementes by following the below steps  

```bash
sudo yum Install awscli -y
```
```bash
yum Install awscli -y
```
```bash
yum update
```
```bash
aws configure
```
```bash
sudo yum Install git -y
```
```bash
yum install awscli -y
```
```bash
git --version
```
## Requirements:  

- ***Python:*** Version 2.7.12 or newer is recommended. Ensure you have the Jinja2 (2.8) and netaddr modules installed.  
- ***Ansible:*** Version 2.1.0.0 or newer is required for the automation scripts.  
- ***cfssl and cfssljson:*** These tools from CloudFlare are required for certificate generation. Download and installation instructions can be found at Cloudflare's GitHub repository.  
- ***Kubernetes CLI (kubectl):*** Needed to interact with the Kubernetes cluster.  
- ***SSH Agent:*** Used for managing SSH keys and connections.  
- ***AWS CLI (optional):*** Useful for managing AWS resources directly.
  
## Installation Guides    

- ***Terraform:*** Install Terraform
```bash
sudo yum install -y yum-utils
```
```bash
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
```
```bash
sudo yum -y install terraform
```
```bash
terraform -help
```
- ***Jinja2:*** Install Python and use pip to install required modules:
```bash
sudo yum update -y
```
```bash
sudo yum install python3 -y
```
```bash
pip3 install jinja2 netaddr boto boto3
```
- ***Ansible:*** Install Ansible
```bash
sudo yum update -y
```
```bash
sudo amazon-linux-extras install epel -y
```
```bash
sudo yum install ansible -y
```
```bash
ansible --version
```
- ***cfssl and cfssljson:*** Follow the instructions on the GitHub page.
```bash
curl -s -L -o /usr/local/bin/cfssl https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssl_1.6.1_linux_amd64
curl -s -L -o /usr/local/bin/cfssljson https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssljson_1.6.1_linux_amd64
chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson
cfssl version
```
- ***Kubernetes CLI:*** Install and Set Up kubectl
```bash
VERSION=$(curl -s https://storage.googleapis.com/kubernetes-release/release/v1.3.10
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.3.10/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version --client
kubectl version 
```


## AWS Setup:
### AWS Credentials and KeyPair
***1. AWS KeyPair:*** Ensure you have an AWS KeyPair for SSH access to instances. This requires a .pem file and the corresponding public key. AWS KeyPair import instructions can be found in the AWS Documentation.

***2. Environment Variables:*** Terraform and Ansible use AWS credentials set as environment variables. Set these in your terminal:

```bash
export AWS_ACCESS_KEY_ID="your_access_key_id"
export AWS_SECRET_ACCESS_KEY="your_secret_access_key"
```
For AWS CLI usage, also set:
```bash
export AWS_DEFAULT_REGION="desired_aws_region"
```
***3. SSH Agent:*** Add your PEM key to the SSH agent for seamless SSH connections:
```bash
ssh-add /path/to/your-keypair.pem
```
### Configuring the Environment
Create a terraform.tfvars file in the ./terraform directory with the following content:
```bash
default_keypair_public_key = "ssh-rsa AAA...your_public_key...zzz"
control_cidr = "your_control_machine_ip/32"
default_keypair_name = "your_keypair_name"
vpc_name = "YourVPCName"
elb_name = "YourELBName"
owner = "YourName"
```

- control_cidr: Your control machine's IP in CIDR notation, ensuring secure access.
- default_keypair_public_key: Your AWS KeyPair's public key.
- Additional variables like vpc_name, elb_name, and owner can be customized as needed.  
  

### AWS Region Configuration
To change the AWS region from the default (eu-west-1), set the region, zone, and default_ami variables in your Terraform configuration. Also, adjust the Ansible hosts/ec2.ini file to match your chosen region.

## Provisioning with Terraform
Navigate to the ./terraform directory and execute:

```bash
terraform plan
terraform apply
```
Terraform will output the Kubernetes API's DNS name and the worker nodes' public IPs. Keep these for later use.  

```
Apply complete! Resources: 12 added, 2 changed, 0 destroyed.
  ...
Outputs:

  kubernetes_api_dns_name = lorenzo-kubernetes-api-elb-1566716572.eu-west-1.elb.amazonaws.com
  kubernetes_workers_public_ip = 54.171.180.238,54.229.249.240,54.229.251.124
```

You will need them later (you may show them at any moment with `terraform output`).

## Generated SSH config

Terraform generates `ssh.cfg`, SSH configuration file in the project directory.
It is convenient for manually SSH into machines using node names (`controller0`...`controller2`, `etcd0`...`2`, `worker0`...`2`), but it is NOT used by Ansible.

e.g.
```
$ ssh -F ssh.cfg worker0
```

## Kubernetes Setup with Ansible
Move to the ./ansible directory to start the Kubernetes setup.

### Cluster Installation
To install Kubernetes components and set up the etcd cluster, run:

```bash
ansible-playbook infra.yaml
```

### Configuring kubectl
Set your Kubernetes API endpoint (provided by Terraform) in the Kubernetes CLI:

```bash
ansible-playbook kubectl.yaml --extra-vars "kubernetes_api_endpoint=your_api_endpoint"
```
Verify the setup with:

```bash
kubectl get componentstatuses
kubectl get nodes
```
Verify all components and minions (workers) are up and running, using Kubernetes CLI (`kubectl`).

```
$ kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}

$ kubectl get nodes
NAME                                       STATUS    AGE
ip-10-43-0-30.eu-west-1.compute.internal   Ready     6m
ip-10-43-0-31.eu-west-1.compute.internal   Ready     6m
ip-10-43-0-32.eu-west-1.compute.internal   Ready     6m
```

### Pod Networking
Configure inter-pod routing:
```bash
ansible-playbook kubernetes-routing.yaml
```

### Deploying a Test Service
Deploy an nginx service to test the cluster:

```bash
ansible-playbook kubernetes-nginx.yaml
```
Check the deployment:

```bash
kubectl get pods -o wide
kubectl get svc nginx --output=json
```
Access the nginx service using one of the worker node's public IP and the exposed port.
Verify pods and service are up and running.

```
$ kubectl get pods -o wide
NAME                     READY     STATUS    RESTARTS   AGE       IP           NODE
nginx-2032906785-9chju   1/1       Running   0          3m        10.200.1.2   ip-10-43-0-31.eu-west-1.compute.internal
nginx-2032906785-anu2z   1/1       Running   0          3m        10.200.2.3   ip-10-43-0-30.eu-west-1.compute.internal
nginx-2032906785-ynuhi   1/1       Running   0          3m        10.200.0.3   ip-10-43-0-32.eu-west-1.compute.internal

> kubectl get svc nginx --output=json
{
    "kind": "Service",
    "apiVersion": "v1",
    "metadata": {
        "name": "nginx",
        "namespace": "default",
...
```

Retrieve the port *nginx* has been exposed on:

```
$ kubectl get svc nginx --output=jsonpath='{range .spec.ports[0]}{.nodePort}'
32700
```

Now you should be able to access *nginx* default page:
```
$ curl http://<worker-0-public-ip>:<exposed-port>
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

The service is exposed on all Workers using the same port (see Workers public IPs in Terraform output).


### Simplifications and Limitations
This setup includes several simplifications not suitable for a production environment, such as the lack of advanced networking configurations, direct SSH access without a bastion host, and minimal security configurations. For a detailed list of these limitations, refer to the provided guide.

By following this guide, you'll have a basic Kubernetes cluster set up on AWS using Terraform and Ansible, suitable for learning and experimentation.
