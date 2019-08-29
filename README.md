# sandbox-lab
Sample HA k8s cluster:

aws s3api create-bucket \
    --bucket sandbox-mnet-dev-state-store \
    --region ca-central-1 \
    --create-bucket-configuration LocationConstraint=ca-central-1


//Versionning to allow for restore
aws s3api put-bucket-versioning --bucket sandbox-mnet-dev-state-store --versioning-configuration Status=Enabled


//Set encryption
aws s3api put-bucket-encryption --bucket sandbox-mnet-dev-state-store --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'


//Create the store

//Prep ENV
export NAME=sandbox.k8s.local
export KOPS_STATE_STORE=s3://sandbox-mnet-dev-state-store

//Check zones:
aws ec2 describe-availability-zones --region ca-central-1

{
    "AvailabilityZones": [
        {
            "State": "available",
            "Messages": [],
            "RegionName": "ca-central-1",
            "ZoneName": "ca-central-1a",
            "ZoneId": "cac1-az1"
        },
        {
            "State": "available",
            "Messages": [],
            "RegionName": "ca-central-1",
            "ZoneName": "ca-central-1b",
            "ZoneId": "cac1-az2"
        }
    ]
}


//Create cluster configuration
kops create cluster $NAME \
    --master-count 3 \
    --node-count 3 \
    --zones ca-central-1a,ca-central-1b \
    --master-zones ca-central-1a \
    --node-size c5.large \
    --node-volume-size 256 \
    --master-size c5.large \
    --api-loadbalancer-type internal \
    --topology private \
    --networking calico \
    --cloud "aws" \
    --ssh-public-key ~/.ssh/magiknet-aws.pub \
    --vpc vpc-665f310e \
    --dry-run \
    -o yaml > $NAME.yaml

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
