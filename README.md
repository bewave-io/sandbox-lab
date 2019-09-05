# sandbox-lab
Sample HA k8s cluster:
```bash
aws s3api create-bucket \
    --bucket sandbox-tools-mnet-dev-state-store \
    --region ca-central-1 \
    --create-bucket-configuration LocationConstraint=ca-central-1


# Versionning to allow for restore
aws s3api put-bucket-versioning --bucket sandbox-tools-mnet-dev-state-store --versioning-configuration Status=Enabled


# Set encryption
aws s3api put-bucket-encryption --bucket sandbox-tools-mnet-dev-state-store --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'


# Create the store

# Prep ENV
export NAME=sandbox-tools.k8s.local
export KOPS_STATE_STORE=s3://sandbox-tools-mnet-dev-state-store

# Check zones:
aws ec2 describe-availability-zones --region ca-central-1

# Create cluster configuration
kops create cluster $NAME \
    --master-count 3 \
    --node-count 3 \
    --zones ca-central-1a,ca-central-1b \
    --master-zones ca-central-1a \
    --node-size m5a.large \
    --node-volume-size 64 \
    --master-size t3a.large \
    --master-volume-size 64 \
    --api-loadbalancer-type internal \
    --topology private \
    --networking calico \
    --cloud "aws" \
    --ssh-public-key ~/.ssh/magiknet-aws.pub \
    --vpc vpc-665f310e \
    --dry-run \
    -o yaml > $NAME.yaml


---

apiVersion: kops/v1alpha2
kind: InstanceGroup
metadata:
  creationTimestamp: null
  labels:
    kops.k8s.io/cluster: sandbox-tools.k8s.local
  name: stores
spec:
  image: kope.io/k8s-1.12-debian-stretch-amd64-hvm-ebs-2019-06-21
  machineType: m5a.large
  maxSize: 3
  minSize: 3
  nodeLabels:
    kops.k8s.io/instancegroup: stores
  role: Node
  rootVolumeSize: 64
  subnets:
  - ca-central-1a
  - ca-central-1b




kops create -f $NAME.yaml
kops create secret --name $NAME sshpublickey admin -i ~/.ssh/rsa_id.pub

# Build the cluster
kops update cluster $NAME --yes
kops rolling-update cluster $NAME --yes

# Validate cluster
kubectl get nodes

kops validate cluster

kubectl -n kube-system get po

# Install Tiller/Helm
https://helm.sh/docs/using_helm/
$ curl -LO https://git.io/get_helm.sh
$ chmod 700 get_helm.sh
$ ./get_helm.sh

kubectl -n kube-system create serviceaccount tiller
kubectl create clusterrolebinding tiller \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:tiller
  
helm init --service-account tiller --history-max 200
# Test tiller install
kubectl -n kube-system  rollout status deploy/tiller-deploy

helm version

# Install Loadbalancer and NginxIngress
helm install stable/nginx-ingress --name main-loadbalancer -f affinity-values.yaml --set rbac.create=true


#certmanager
# Install the CustomResourceDefinition resources separately
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.9/deploy/manifests/00-crds.yaml

# Create the namespace for cert-manager
kubectl create namespace cert-manager

# Label the cert-manager namespace to disable resource validation
kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
# Sample values: https://github.com/jetstack/cert-manager/blob/master/deploy/charts/cert-manager/values.yaml
helm install \
  -f affinity-values.yaml \
  --name cert-manager \
  --namespace cert-manager \
  --version v0.9.1 \
  jetstack/cert-manager

#Remove with
helm del --purge cert-manager

#Validate with
kubectl get pods --namespace cert-manager

---
Creating issuers:

- Staging:
    kubectl apply -f staging-issuer.yaml
  
- Production:
    kubectl apply -f production-issuer.yaml
  
---
Create ingress 

    kubectl apply -f ingress.yaml


# Cloud config addendum for OpenEBS
####################################################################################

echo "== preparing iscsi for bewave=="
set -x
date
apt-get update
apt-get install xfsprogs
mkfs -t xfs /dev/nvme1n1
mkdir /mnt/blabla
mount /dev/nvme1n1 /mnt/blabla
sh -c 'echo "/dev/nvme1n1 /mnt/blabla auto defaults,nofail,comment=cloudconfig 0 2" >> /etc/fstab'
apt-get -y install open-iscsi
 grep "@reboot root sleep 120;service open-iscsi restart" /etc/crontab || sudo sh -c 'echo "@reboot root sleep 120;service open-iscsi restart" >> /etc/crontab'
systemctl enable open-iscsi
echo "== preparing iscsi for bewave DONE=="
reboot




```
