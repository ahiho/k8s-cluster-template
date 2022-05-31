# Cluster configs for k8s-cluster-template

This repos contains stuffs for setup new k8s cluster on AWS using kops and fixed version of kops to avoid big break change in case we have to update cluster after a white

## Requirement
- kubectl
- aws cli

## Cluster information
- kops version: 1.23.1
- Kubernetes version: v1.23.5
- Ingress Nginx version: v1.2.0
- Cluster Autoscaler version: v1.23.0
- Cert manager version: v1.8.0
- Kubectl version: v1.23.6

# Getting started

## 1. Install kops
If you already have kops installed and same with working version, let's ignore this.

Run this to install version of kops when cluster created

```
curl -Lo ./bin/kops https://github.com/kubernetes/kops/releases/download/v1.23.1/kops-darwin-amd64
chmod +x ./bin/kops
```
In case your `kubectl` version is difference with version in cluster information, you can download special version by this command

```
curl -Lo ./bin/kubectl https://dl.k8s.io/release/v1.23.6/bin/darwin/amd64/kubectl
chmod +x ./bin/kubectl
```

## 2. Create cluser

### Deploy cluster

Create `.env` file from `.env.example` with correct value

Create ssh key for new cluster if needed

Run bellow scripts to create file `kops/cluster.yaml` (replace `k8s_template.pub` witn with name of public key file)
```
source .env

./bin/kops create cluster --name=$NAME --state=s3://$KOPS_STATE_STORE --cloud=aws --zones=$ZONES --ssh-public-key ./k8s_template.pub --dry-run -oyaml > kops/cluster.yaml
```

Edit value of files in `kops` depend on what you wanna create

Create clusters with bellow commands
```
kops create -f kops/cluster.yaml
# bellow command depend on files from your cluster
# ./bin/kops create -f kops/node-ondemand-1.yaml
# ./bin/kops create -f kops/node-spot-1.yaml
```

Export kubeconfig for admin account

```
kops export kubeconfig $NAME --admin=87600h0m0s
```

### Install components

Replace variable with correct value

* replace `<NAME>` in `cluster-autoscaler-autodiscover.yaml` with name of cluster
* replace `<AWS_REGION>` in `cluster-autoscaler-autodiscover.yaml` with region
* replace `<ACME_EMAIL>` in `prod_issuer.yaml` with email for certbot


Follow https://kops.sigs.k8s.io/getting_started/aws/ to create aws user with correct permissions

Create another aws user with bellow permissions for Autoscaler
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeTags",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeLaunchTemplateVersions"
      ],
      "Resource": ["*"]
    },
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeInstanceTypes",
        "eks:DescribeNodegroup"
      ],
      "Resource": ["*"]
    }
  ]
}
```

Replace `BASE64_OF_YOUR_AWS_ACCESS_KEY_ID` and `BASE64_OF_YOUR_AWS_SECRET_ACCESS_KEY` with based 64 of aws api key and secret for Autoscaler


After cluster create success, install some important components for cluster by this command
```
kubectl apply -f kubectl/asg-secret.yaml # Add secret for Auto scaling group
kubectl apply -f kubectl/cluster-autoscaler-autodiscover.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.2.0/deploy/static/provider/aws/deploy.yaml
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.0/cert-manager.yaml
kubectl apply -f kubectl/prod_issuer.yaml
```
### Update information for specs
Update this `README.MD` with your kops and kubectl version

## 3. Update cluster

Using `kops` version in this `README.MD` will help avoid newer version make too many change and can cause system down for a while

# Change logs

Append change logs here 