# Crossplane Setup 

## Pre-req
Get the cluster ID
```
CLUSTERID=$(oc get infrastructure cluster -o json | jq -r .status.infrastructureName)
CLUSTERTAG="kubernetes.io/cluster/$CLUSTERID"
```
Create crossplane namespace
```
oc new-project crossplane
```

## Install Operator 
Install Crossplane operator via Operator Hub
_note_ : install operator to `crossplane` namespace

## Create an AWS Crossplane Cluster Package
Create AWS Provider 
```
oc apply -f deploy/crossplane/cluster_package_aws.yaml
```

## Create AWS IAM creds
Configure AWS Creds
```
# retrieve profile's credentials, save it under 'default' profile, and base64 encode it
aws_profile=default
BASE64ENCODED_AWS_ACCOUNT_CREDS=$(echo -e "[default]\naws_access_key_id = $(aws configure get aws_access_key_id --profile $aws_profile)\naws_secret_access_key = $(aws configure get aws_secret_access_key --profile $aws_profile)" | base64  | tr -d "\n")
# retrieve the profile's region from config
AWS_REGION=$(aws configure get region --profile ${aws_profile})
```
```
cat > provider.yaml <<EOF
---
apiVersion: v1
kind: Secret
metadata:
  name: aws-account-creds
  namespace: crossplane
type: Opaque
data:
  credentials: ${BASE64ENCODED_AWS_ACCOUNT_CREDS}
---
apiVersion: aws.crossplane.io/v1alpha3
kind: Provider
metadata:
  name: aws-provider
spec:
  region: ${AWS_REGION}
  credentialsSecretRef:
    namespace: crossplane
    name: aws-account-creds
    key: credentials
EOF
```

apply it to the cluster:
```
oc apply -f provider.yaml
```

delete the credentials variable
```
unset BASE64ENCODED_AWS_ACCOUNT_CREDS
```

## Setup VPC
```
oc apply -f deploy/network/vpc.yaml
```
verify vpc has been created 
```
aws ec2 describe-vpcs --filters Name=tag-key,Values=mase/crossplane | jq
```

## Setup Security Group
Get the Cluster VPC CIDR Block
```
aws ec2 describe-vpcs --filters Name=tag-key,Values=$CLUSTERTAG | jq
```
Update the security group such that it contains the cluster vpc cidr
```
oc apply -f deploy/network/security_group.yaml
```
verify security group has been created
```
aws ec2 describe-security-groups --filters Name=tag-key,Values=mase/crossplane | jq
```

## Create Subnets
```
oc apply -f deploy/network/subnets.yaml
```
verify subnets have been created
```
aws ec2 describe-subnets --filters Name=tag-key,Values=mase/crossplane | jq
```

## Create Subnet Group
Create subnet group for RDS instances
```
oc apply -f deploy/postgres/subnet_group_rds.yaml
```

verify subnet group has been created
```
aws rds describe-db-subnet-groups --db-subnet-group-name mase-rds-sg | jq
```

Create subnet group for Elasticache instances, first get the subnet id's from the subnets created in a previous step
```
aws ec2 describe-subnets --filters Name=tag-key,Values=mase/crossplane | grep SubnetId 
```

Update `deploy/redis/subnet_group_cache.yaml` with the output, and apply the cache subnet group
```
oc apply -f deploy/redis/subnet_group_cache.yaml
```

verify elasticache subnet group has been created
```
aws elasticache describe-cache-subnet-groups --cache-subnet-group-name mase-elasticache-sg
```

## VPC Peering
With the networking resources provisioned we need to manually configure the routes and VPC peering
- Create Peering Connection between crossplane vpc and cluster vpc
  - Select the crossplane vpc as requester
  - Select the cluster vpc as accepter
- Accept Peering Connection
  - Highlight the peering connection
  - Select actions
  - Click accept peering connection
- Add routes to VPC route tables 
  - To send and receive traffic across this VPC peering connection, you must add a route to the peered VPC in one or more of your VPC route tables.

## Provision RDS
Create a class
```
oc apply -f deploy/postgres/rds_class.yaml
```
Create the RDS instance via an RDS Claim
```
oc apply -f deploy/postgres/rds_claim.yaml
```

## Provision Elasticache
```
oc apply -f deploy/redis/cache_replication_group.yaml
```

## Provision S3 Bucket
Create a class
```
oc deploy -f deploy/storage/s3_bucket_class.yaml
```
Create a claim
```
oc deploy -f deploy/storage/s3_bucket_claim.yaml
```
