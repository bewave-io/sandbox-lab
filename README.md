# sandbox-lab
Sample HA k8s cluster:
```bash
aws s3api create-bucket \
    --bucket sandbox-tools-mnet-dev-state-store \
    --region ca-central-1 \
    --create-bucket-configuration LocationConstraint=ca-central-1


//Versionning to allow for restore
aws s3api put-bucket-versioning --bucket sandbox-tools-mnet-dev-state-store --versioning-configuration Status=Enabled


//Set encryption
aws s3api put-bucket-encryption --bucket sandbox-tools-mnet-dev-state-store --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'


//Create the store

//Prep ENV
export NAME=sandbox-tools.k8s.local
export KOPS_STATE_STORE=s3://sandbox-tools-mnet-dev-state-store

//Check zones:
aws ec2 describe-availability-zones --region ca-central-1

//Create cluster configuration
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
kops update cluster $NAME --yes
kops rolling-update cluster $NAME --yes

//Build the cluster
kops update cluster ${NAME} --yes

//Validate cluster
kubectl get nodes

kops validate cluster

kubectl -n kube-system get po
```
