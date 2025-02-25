# Kubernetes setup using Kubernetes Operations (kOps) on Amazon EC2

Here are few resources and steps to get you started:

You'll learn how to:

1. Create a kubernetes cluster
2. Deploy app
3. Explore app
4. Upgrade cluster
5. Scaling cluster
6. Terminate cluster

## Create an Ubuntu EC2 instance or you can make use of your personal infrastructure

**Prerequisites**:

1. AWS subscription
2. AWS command-line interface (CLI)
3. Python3
4. kOps
5. Kube command-line interface (Kubectl)

# Setup your environment

## Install dependencies

> [!NOTE]
> The command <mark>sudo apt-get update</mark> downloads package information from all configured sources.

```python
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release software-properties-common
```

- Install the Apt repository signing keys, using the following commands:

[Kubernetes](https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/)

> [!NOTE]
> Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories, so you can disregard the version in the URL:

```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

> [!NOTE]
> The command prints out the line describing the relevant repository and this is piped to <mark>sudo tee</mark> as a way of writing to <mark>/etc/apt/sources.list.d/kubernetes.list</mark>. Replace the <mark>apt</mark> repository definition so that <mark>apt</mark> points to the new repository instead of the Google-hosted repository. Make sure to replace the Kubernetes minor version in the command below with the minor version you're currently using.

```
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

- Install the Python, AWS CLI, Kubectl workloads, using the following commands:

> [!NOTE]
> The command <mark>sudo apt-get update</mark> downloads package information from all configured sources.

```python
sudo apt-get update
sudo apt-get install -y python3-pip python3-venv kubectl
```

## Verify the installation

```python
which python3
pip3 --version
kubectl version
```

The command installs AWS CLI <mark>latest/specific</mark> version, using pip3.

- For an externally managed environment, create and activate a virtual environment:

```python
python3 -m venv .venv
source .venv/bin/activate
```

- To get the latest version of the AWS CLI using pip3.

```python
pip3 install awscli --upgrade
```

- To get the <mark>specific version</mark> of the AWS CLI using pip3.

```python
pip3 install awscli<1.6.312 --upgrade
```

[AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html)

To download, and install AWS CLI <mark>latest/specific version</mark>, using curl.

```sh
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

To update AWS CLI <mark>latest/specific version</mark>, using curl.

```sh
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install --bin-dir /usr/local/bin --install-dir /usr/local/aws-cli --update
```

- To install AWS CLI v2 <mark>latest version</mark>, using curl.

```sh
curl -o awscliv2.sig https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip.sig
```

- To install AWS CLI v2 <mark>specific version</mark>, using curl.

```sh
curl -o awscliv2.sig https://awscli.amazonaws.com/awscli-exe-linux-x86_64-2.0.30.zip.sig
```

## Verify AWS CLI v2 installation

```shell
which aws
aws --version
```

## Set the path environment for the current session

```shell
export PATH="$PATH:/home/ubuntu/.local/bin/"
```

## Configure the AWS CLI

- Provide the below permissions to your IAM user. If you're using the admin user, the below permissions are available by default.

1. AmazonEC2FullAccess
2. AmazonS3FullAccess
3. IAMFullAccess
4. AmazonVPCFullAccess
5. AmazonSQSFullAccess
6. AmazonEventBridgeFullAccess
7. AmazonRoute53FullAccess

- Create an IAM user/role with Route53, EC2, IAM and S3 full access

```shell
# create setup dedicated user and group
aws iam create-group --group-name kops

aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonSQSFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEventBridgeFullAccess --group-name kops

aws iam create-user --user-name kops

aws iam add-user-to-group --user-name kops --group-name kops

aws iam create-access-key --user-name kops
```

- Verify that user and group exists.

```shell
aws iam list-users

aws iam list-groups-for-user --user-name=kops --query='Groups[].GroupName'
[
    "kops"
]
```

- Configure the aws client to use your new IAM user.

```shell
aws configure
#AWS Access Key ID [None]: accesskey
#AWS Secret Access Key [None]: secretkey
#Default region name [None]: us-east-2
#Default output format [None]:
```

```shell
aws configure [--profile profile-name]
```

> [!NOTE]
> If the command is run with no arguments, you will be prompted for configuration values such as your AWS Access Key id and your AWS Secret Access Key. You can configure a named profile using the --profile argument. If your config file does not exist (default location is \~/.aws/config), the AWS CLI will create it for you. The values you provide will be written to the shared credentials file (~/.aws/credentials).

## Install kOps (using curl or wget)

```shell
curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
chmod +x kops-linux-amd64
sudo mv kops-linux-amd64 /usr/local/bin/kops
```

OR

```shell
wget https://github.com/kubernetes/kops/releases/download/v1.30.0/kops-linux-amd64
sudo mv kops-linux-amd64 /usr/local/bin/kops
sudo chmod +x /usr/local/bin/kops
```

## Verify kOps

```shell
which kops
kops version
```

## Create a Route53 private hosted zone (you can create Public hosted zone if you have a domain)

## Setup AWS S3 bucket for kOps to store cluster state objects

In order to store the state of your cluster, and the representation of your cluster, we need to create a dedicated S3 bucket for <mark>kOps</mark> to use. The following <mark>create-bucket</mark> command creates a bucket named kops-ashu-storage

> [!TIP]
> It is recommended keeping the creation of this bucket confined to us-east-1, otherwise more work will be required.

```shell
aws s3api create-bucket \
  --bucket kops-ashu-storage \
  --region us-east-1
```

The following <mark>create-bucket</mark> command creates a bucket named kops-ashu-storage that uses the bucket owner enforced setting for S3 Object Ownership.

```shell
aws s3api create-bucket \
  --bucket kops-ashu-storage \
  --region us-east-1 \
  --object-ownership BucketOwnerEnforced
```

The following <mark>create-bucket</mark> creates a bucket outside the us-east-1 region:

```shell
aws s3api create-bucket \
  --bucket kops-ashu-storage \
  --region eu-west-1 \
  --create-bucket-configuration LocationConstraint=eu-west-1
```

> [!NOTE]
> S3 requires <mark>--create-bucket-configuration LocationConstraint=<region></mark> for regions other than <mark>us-east-1</mark>.

> [!NOTE]
> It is **_`strongly`_** recommended **_versioning_** your S3 bucket in case you ever need to revert or recover a previous state store.

```shell
aws s3api put-bucket-versioning \
  --bucket kops-ashu-storage \
  --versioning-configuration Status=Enabled
```

## Expose the state store environment variable

```shell
export KOPS_STATE_STORE=s3://kops-ashu-storage
```

## Create the kubernetes cluster using kOps

Creates a Kubernetes cluster using command line flags. This command creates cloud based resources such as networks and virtual machines. Once the infrastructure is in place Kubernetes is installed on the virtual machine. These operations are done in parallel and rely on eventual consistency.

`kops create cluster [CLUSTER] [flags]`

```
  # Create a cluster in AWS in a single zone.
  kops create cluster
    --name=k8s-cluster.example.com \
    --state=s3://my-state-store \
    --zones=us-east-1a \
    --node-count=2

  # Create a cluster in AWS with a High Availability control plane. This cluster has also been configured for private networking in a kops-managed VPC. The bastion flag is set to create an entrypoint for admins to SSH.
  export KOPS_STATE_STORE="s3://my-state-store"
  export CONTROL_PLANE_SIZE="c5.large"
  export NODE_SIZE="m5.large"
  export ZONES="us-east-1a,us-east-1b,us-east-1c"
  kops create cluster k8s-cluster.example.com \
    --node-count 3 \
    --zones $ZONES \
    --node-size $NODE_SIZE \
    --control-plane-size $CONTROL_PLANE_SIZE \
    --control-plane-zones $ZONES \
    --networking cilium \
    --topology private \
    --bastion="true" \
    --yes

  # Create a cluster in Digital Ocean.
  export KOPS_STATE_STORE="do://my-state-store"
  export ZONES="NYC1"
  kops create cluster k8s-cluster.example.com \
    --cloud digitalocean \
    --zones $ZONES \
    --control-plane-zones $ZONES \
    --node-count 3 \
    --yes

  # Generate a cluster spec to apply later. Run the following, then: kops create -f filename.yaml
  kops create cluster --name=k8s-cluster.example.com \
    --state=s3://my-state-store \
    --zones=us-east-1a \
    --node-count=2 \
    --dry-run \
    -oyaml > filename.yaml
```

> Options:

```shell
  --admin-access strings                    Restrict API access to this CIDR.  If not set, access will not be restricted by IP. (default [0.0.0.0/0,::/0])
  --api-loadbalancer-type string            Type of load balancer for the Kubernetes API: public or internal
  --api-public-name string                  Domain name of the public Kubernetes API
  --api-ssl-certificate string              ARN of the SSL Certificate to use for the Kubernetes API load balancer (AWS only)
  --associate-public-ip                     Specify --associate-public-ip=[true|false] to enable/disable association of public IP for control-plane ASG and nodes. Default is 'true'.
  --authorization string                    Authorization mode: AlwaysAllow or RBAC (default "RBAC")
  --bastion                                 Enable a bastion instance group. Only applies to private topology.
  --bastion-image string                    Machine image for bastions. Takes precedence over --image
  --channel string                          Channel for default versions and configuration to use (default "stable")
  --cloud string                            Cloud provider to use - aws, digitalocean, gce, hetzner, openstack
  --cloud-labels string                     A list of key/value pairs used to tag all instance groups (for example "Owner=John Doe,Team=Some Team").
  --control-plane-count int32               Number of control-plane nodes. Defaults to one control-plane node per control-plane-zone
  --control-plane-image string              Machine image for control-plane nodes. Takes precedence over --image
  --control-plane-security-groups strings   Additional pre-created security groups to add to control-plane nodes.
  --control-plane-size strings              Machine type(s) for control-plane nodes
  --control-plane-tenancy string            Tenancy of the control-plane group (AWS only): default or dedicated
  --control-plane-volume-size int32         Instance volume size (in GB) for control-plane nodes
  --control-plane-zones strings             Zones in which to run control-plane nodes. (must be an odd number)
  --disable-subnet-tags                     Disable automatic subnet tagging
  --discovery-store string                  A public location where we publish OIDC-compatible discovery information under a cluster-specific directory. Enables IRSA in AWS.
  --dns string                              DNS type to use: public, private, none
  --dns-zone string                         DNS hosted zone (defaults to longest matching zone)
  --dry-run                                 If true, only print the object that would be sent, without sending it. This flag can be used to create a cluster YAML or JSON manifest.
  --encrypt-etcd-storage                    Generate key in AWS KMS and use it for encrypt etcd volumes
  --etcd-clusters strings                   Names of the etcd clusters: main, events (default [main,events])
  --etcd-storage-type string                The default storage type for etcd members
  --gce-service-account string              Service account with which the GCE VM runs. Warning: if not set, VMs will run as default compute service account.
  -h, --help                                    help for cluster
  --image string                            Machine image for all instances
  --instance-manager string                 Instance manager to use (cloudgroups or karpenter. Default: cloudgroups) (default "cloudgroups")
  --ipv6                                    Use IPv6 for the pod network (AWS only)
  --kubernetes-feature-gates strings        List of Kubernetes feature gates to enable/disable
  --kubernetes-version string               Version of Kubernetes to run (defaults to version in channel)
  --network-cidr strings                    Network CIDR(s) to use
  --network-id string                       Shared Network or VPC to use
  --networking string                       Networking mode.  kubenet, external, flannel-vxlan (or flannel), flannel-udp, calico, canal, kube-router, amazonvpc, cilium, cilium-etcd, cni. (default "cilium")
  --node-count int32                        Total number of worker nodes. Defaults to one node per zone
  --node-image string                       Machine image for worker nodes. Takes precedence over --image
  --node-security-groups strings            Additional pre-created security groups to add to worker nodes.
  --node-size strings                       Machine type(s) for worker nodes
  --node-tenancy string                     Tenancy of the node group (AWS only): default or dedicated
  --node-volume-size int32                  Instance volume size (in GB) for worker nodes
  --os-dns-servers string                   comma separated list of DNS Servers which is used in network
  --os-ext-net string                       External network to use with the openstack router
  --os-ext-subnet string                    External floating subnet to use with the openstack router
  --os-kubelet-ignore-az                    Attach volumes across availability zones
  --os-lb-floating-subnet string            External subnet to use with the Kubernetes API
  --os-network string                       ID of the existing OpenStack network to use
  --os-octavia                              Use octavia load balancer API
  --os-octavia-provider string              Octavia provider to use
  --out string                              Path to write any local output
  -o, --output string                           Output format. One of json or yaml. Used with the --dry-run flag.
  --project string                          Project to use (must be set on GCE)
  --set strings                             Directly set values in the spec (default [])
  --ssh-access strings                      Restrict SSH access to this CIDR.  If not set, uses the value of the admin-access flag.
  --ssh-public-key string                   SSH public key to use
  --subnets strings                         Shared subnets to use
  --target string                           Valid targets: direct, terraform. Set this flag to terraform if you want kOps to generate terraform (default "direct")
  -t, --topology string                         Network topology for the cluster: 'public' or 'private'. Defaults to 'public' for IPv4 clusters and 'private' for IPv6 clusters.
  --unset strings                           Directly unset values in the spec
  --utility-subnets strings                 Shared utility subnets to use
  -y, --yes                                     Specify --yes to immediately create the cluster
  --zones strings                           Zones in which to run the cluster
```

Options inherited from parent commands:

```shell
  --config string   yaml config file (default is $HOME/.kops.yaml)
  --name string     Name of cluster. Overrides KOPS_CLUSTER_NAME environment variable
  --state string    Location of state storage (kops 'config' file). Overrides KOPS_STATE_STORE environment variable
  -v, --v Level         number for the log level verbosity
```

Create kubernetes cluster definitions on S3 bucket

- Create cluster (bare minimum configurations: **_recommended_**)

```shell
kops create cluster \
  --name=demok8scluster.k8s.local \
  --cloud=aws \
  --state=s3://kops-ashu-storage \ #flag if environment variable KOPS_STATE_STORE not set for the current session.
  --zones=us-west-2a
```

- Create cluster (pre-defined configurations)

```shell
kops create cluster \
  --name=demok8scluster.k8s.local \
  --cloud=aws \
  --state=s3://kops-ashu-storage \ #flag if environment variable KOPS_STATE_STORE not set for the current session.
  --zones=us-west-2a \
  --node-count=1 \
  --node-size=t2.micro \
  --control-plane-size=t2.micro \
  --control-plane-volume-size=8 \
  --node-volume-size=8
```

[Resources list](https://github.com/ashuhatkar/ashulearn-kubernetes-setup-on-prod-systems/blob/develop/01-install/kops-provision-resources.md)

## Review/customize cluster configuration

**Now we have a cluster configuration**, we can look at every aspect that defines our cluster by editing the description.

```
kops edit cluster --name demok8scluster.k8s.local
```

This opens your editor (as defined) and allows you to edit the configuration. The configuration is loaded from the S3 bucket we created earlier, and automatically updated when we save and exit the editor.

## Build/run kubernetes cluster

```shell
# This start an actual provisioning process; be ready to start paying for the resources
kops update cluster --name demok8scluster.k8s.local --yes --admin
```

> [!NOTE]
> This'll take a while. Once it finishes you'll have to wait longer while the booted instances finish downloading Kubernetes components and reach a "ready" state.

## Validate cluster

Kubernetes usage

```shell
kops get cluster
kops validate cluster --name demok8scluster.k8s.local
kubectl version

# Review nodes; they are just EC2 instance
kubectl get nodes
kubectl get sc
kubectl get ns

# Review system pods; pod is a group of tightly coupled Docker containers
kubectl get all -n kube-system
```

## Deploying Nginx container on Kubernetes

## Create a custom Namespace

```shell
kubectl create namespace ns-prod-alpha01
```

## Verify the created namespace

```shell
kubectl get namespaces
```

## Let's deploy an Nginx web server named "mynginx" inside the Namespace

```shell
#kubectl run mynginx --image=nginx --replicas=2 -n ns-prod-alpha01
kubectl create deployment mynginx --image=nginx -n ns-prod-alpha01
```

## Now, let's verify that the "mynginx" Deployment has been created inside the "ns-prod-alpha01" Namespace.

```shell
#kubectl get all -n ns-prod-alpha01
kubectl get deployments -n ns-prod-alpha01
```

## Verify and get pod name.

```shell
kubectl get pods -n ns-prod-alpha01
```

## Access your application via kubernetes feature "port-forward" 8080 locally

```shell
#kubectl port-forward <pod_name> 8080:80 -n <namespace>
kubectl port-forward mynginx-XXXXXXXXXX-z9pp6 8080:80 -n ns-prod-alpha01
```

## Verify in browser

- <http://localhost:8080/>
- <http://127.0.0.1:8080/>

## Delete the sample deployment

```shell
kubectl delete deployment mynginx -n ns-prod-alpha01
```

## Terminate cluster

```sh
# Preview AWS resources that will be terminated with cluster
kops delete cluster --name demok8scluster.k8s.local

# Execute an actual termination
kops delete cluster --name demok8scluster.k8s.local --yes
```
