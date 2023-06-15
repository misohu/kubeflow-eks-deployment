# Install Kubeflow on EKS
This repo contains set of commands for [this](https://youtu.be/EqZTXpHYT7w) YouTube video.

If you want to learn Kubernetes from scratch [here](https://www.udemy.com/course/kubernetes-for-beginners-with-aws-examples/?referralCode=6296632C3AA7FE388626) is my course. Thanks for your support.

# AWS setup
First create AWS account (docs [here](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-creating.html)).

Second install AWS cli. (official documentation [here](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html))
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Next create IAM admin user in the GUI (docs [here](https://docs.aws.amazon.com/IAM/latest/UserGuide/getting-set-up.html)) according to video and make sure you download the credentials csv (how to create access keys [here](https://aws.amazon.com/premiumsupport/knowledge-center/create-access-key/)).

Next create ssh key-pair `eksctl-keypair` in the EC2 console and download the csv.

Next configure the aws cli with the access credentials
```
aws configure
```

Install and setup kubectl 
```
sudo snap install kubectl --classic
mkdir ~/.kube
```

Next install and setup `eksctl`. Official docs [here](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html) 
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

# Cluster setup
Create a cluster with eksctl with two worker nodes and use the keypair
t2.2xlarge $0.3712 x 2 + $0.10 (for controllplane)  hourly = $20,2176 daily 
```
eksctl create cluster --name kubeflow --region eu-central-1 --ssh-access --ssh-public-key=kubeflow-example-keypair --node-type=t2.2xlarge --nodes=2 --node-volume-size=100 --node-volume-type=gp2 --version=1.23
```

EBS CSI setup link [here](https://docs.aws.amazon.com/eks/latest/userguide/managing-ebs-csi.html#adding-ebs-csi-eks-add-on).
Create IAM OpenID connect provider:
```
eksctl utils associate-iam-oidc-provider --cluster kubeflow --approve
```

Create the Amazon EBS CSI driver IAM role for service account
```
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster kubeflow \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-only \
  --role-name AmazonEKS_EBS_CSI_DriverRole
```

Add the Amazon EBS CSI add-on
```
eksctl create addon --name aws-ebs-csi-driver --cluster kubeflow --service-account-role-arn arn:aws:iam::<accountID>:role/AmazonEKS_EBS_CSI_DriverRole --force
```

Sample application
```
git clone https://github.com/kubernetes-sigs/aws-ebs-csi-driver.git
cd aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning/
kubectl apply -f manifests/
kubectl get pods --watch
kubectl get pv
```

Remove sample application
```
kubectl delete -f manifests/
```

# Juju setup
Juju is the Charmed Operator Framework; an open source framework that uses Charmed Operators, or 'Charms’, to deploy cloud infrastructure and applications and manage their operations from Day 0 through Day 2. 

Install Juju
```
sudo snap install juju --classic
```

Register the cluster as a Kubernetes cloud (kubeflow is the name of eksctl cluster)
```
juju add-k8s kubeflow 
```

Bootstrap controller in the cloud 
```
juju bootstrap --no-gui kubeflow kubeflow-controller 
```

Add model (must be named kubeflow in this case)
```
juju add-model kubeflow
```

Now we can check that namespace kubeflow exists 
```
kubectl get ns 
```

# Kubeflow deployment
Now Deploy the kubeflow bundle. Check the latest stable version in the [charmhub](https://charmhub.io/).

```
juju deploy kubeflow --channel=1.6/stable --trust
```

Wait until charms are active 
```
juju status --watch 5s
```

Sometimes there is a waiting for gateway message (in tesorboard) which can be fixed with 
```
juju run --unit istio-pilot/0 -- "export JUJU_DISPATCH_PATH=hooks/config-changed; ./dispatch"
```

Now get the external dns of the only loadbalancer in the kubeflow namespace 
```
kubectl get svc -n kubeflow
```

Configure the dex and oidc with that dns with dashboard credentials 
```
juju config dex-auth public-url=<external_url>; juju config oidc-gatekeeper public-url=<external_url>; juju config dex-auth static-password=user123; juju config dex-auth static-username=user123@email.com
``` 

Go to the dns in your browser

# Cleanup

```
eksctl delete cluster --name=kubeflow
juju unregister kubeflow-controller 
```

# One Command alternative 
Alternatively you can use the `cluster.yaml` file to instantly create the cluster with the storage class in one command. Make sure to change the `publicKeyName` to the public keey which you want to use in the given region. After everything is setup run.

```
eksctl create cluster -f cluster.yaml
```
