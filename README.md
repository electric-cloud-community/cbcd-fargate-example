# CloudBees CD on Fargate example #

## Table of Contents ##

1. [Introduction](#Introduction)
2. [Setup EKS cluster](#setup-eks-cluster)
3. [Setup EFS FileSystem](#setup-efs-filesystem)
4. [Setup MySQL Database in RDS](#setup-mysql-database-in-rds)
5. [Setup ALB](#setup-alb)
6. [Deploy CloudBees CD](#deploy-cloudbees-cd)

## Introduction ##

AWS Fargate has some differences compared to using EC2 instances that need to be taken care of when deploying Cloudbees CD.  These differences include:

- No dynamic provisioning of Persistent Volumes
- Privileged mode not allowed

Below is a set of example instructions that show setting up CloudBees CD on Fargate using an EFS filesystem and a mySQL database running in Amazon RDS.

There are a couple of known issues at the moment:

- DevOps Insight uses elasticsearch, and this currently requires privileged execution.  This is because of the 'max virtual memory areas vm.max_map_count' setting.  Privileged execution is required to execute 'sysctl -w vm.max_map_count=262144' which changes the setting for the host.  Without doing this elasticsearch fails to start with 'max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144] on docker host' error.  Use of sysctl is currently not supported on fargate (https://github.com/aws/containers-roadmap/issues/460).
- Zookeeper has an issue where it is unable to form a quorum.  There is a connection refused error when the nodes attempt to connect to each other.
- Nginx ingress controller also fails due to not having privileged execution.  Using ALB in this example instead.  Sachin Gade tells me that nginx can be installed separately (using it's own helm chart and not installed as part of ours) with the following setting 'podSecurityPolicy.enabled=true' and it should work.  I haven't tested it yet so these instructions use Amazon ALB instead.

The first issue means this example currently does not include a working DevOps Insight component (no reporting and dashboards)
.
Due to the second issue, this example is for a non-clustered set up of the CD-Server component of CloudBees CD.

## Setup EKS cluster ##

```bash
# Set environment variables
CBCD_EKS_CLUSTER=pcherry-cloudbeescd
CBCD_AWS_REGION=us-east-1

# Create eksctl create cluster config file:
eksctl create cluster --name $CBCD_EKS_CLUSTER --region $CBCD_AWS_REGION --fargate --alb-ingress-access --dry-run > clusterconfig.yaml

# Edit clusterconfig.yaml as needed, e.g. to set desired Availability Zones
vi clusterconfig.yaml

# Create the cluster
eksctl create cluster -f clusterconfig.yaml

# Store some details we need later:
CBCD_VPC_ID=$(aws eks describe-cluster --name $CBCD_EKS_CLUSTER --query "cluster.resourcesVpcConfig.vpcId" --region $CBCD_AWS_REGION --output text) && echo $CBCD_VPC_ID
CBCD_CIDR_BLOCK=$(aws ec2 describe-vpcs --vpc-ids $CBCD_VPC_ID --query "Vpcs[].CidrBlock" --region $CBCD_AWS_REGION --output text) && echo $CBCD_CIDR_BLOCK
```

## Setup EFS FileSystem ##

```bash
# Create the filesystem
CBCD_EFS_FS_ID=$(aws efs create-file-system \
  --creation-token $CBCD_EKS_CLUSTER-FS \
  --encrypted \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --tags Key=Name,Value=$CBCD_EKS_CLUSTER-FS \
  --region $CBCD_AWS_REGION \
  --output text \
  --query "FileSystemId") && echo $CBCD_EFS_FS_ID

# Create Security Group:
CBCD_EFS_SG_ID=$(aws ec2 create-security-group \
  --description $CBCD_EKS_CLUSTER \
  --group-name $CBCD_EKS_CLUSTER \
  --vpc-id $CBCD_VPC_ID \
  --region $CBCD_AWS_REGION \
  --query 'GroupId' --output text) && echo $CBCD_EFS_SG_ID

aws ec2 authorize-security-group-ingress \
  --group-id $CBCD_EFS_SG_ID \
  --protocol tcp \
  --port 2049 \
  --cidr $CBCD_CIDR_BLOCK \
  --region $CBCD_AWS_REGION

# Create EFS mount targets:
for subnet in $(aws eks describe-fargate-profile \
  --output text --cluster-name $CBCD_EKS_CLUSTER\
  --fargate-profile-name fp-default  \
  --region $CBCD_AWS_REGION  \
  --query "fargateProfile.subnets"); \
do (aws efs create-mount-target \
  --file-system-id $CBCD_EFS_FS_ID \
  --subnet-id $subnet \
  --security-group $CBCD_EFS_SG_ID \
  --region $CBCD_AWS_REGION); \
done

# Now setup efs-sc StorageClass to use that filesystem.
curl -o storageclass.yaml https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/storageclass.yaml

# Edit storageclass.yaml to use the fs id (the value of $CBCD_EFS_FS_ID).  An example can be found in the repo (storageclass-example.yaml).
vi storageclass.yaml

# Apply the yaml:
kubectl apply -f storageclass.yaml

# The default storageclass is gp2, which doesn't work with Fargate.  Set efs-sc storageclass as the default:
kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
kubectl patch storageclass efs-sc -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

## Setup MySQL Database in RDS ##

```bash
#Get VPC's private subnets:
CBCD_PRIVATE_SUBNETS=$(aws eks describe-fargate-profile \
  --fargate-profile-name fp-default  \
  --cluster-name $CBCD_EKS_CLUSTER\
  --region $CBCD_AWS_REGION \
  --query "fargateProfile.[subnets]" --output text | awk -v OFS="," '{for(i=1;i<=NF;i++)if($i~/subnet/)$i="\"" $i "\"";$1=$1}1') && echo $CBCD_PRIVATE_SUBNETS

# Create a DB subnet group:
aws rds create-db-subnet-group \
  --db-subnet-group-name cbcd-mysql-subnet \
  --subnet-ids "[$CBCD_PRIVATE_SUBNETS]" \
  --db-subnet-group-description "Subnet group for $CBCD_EKS_CLUSTER MySQL RDS." \
  --region $CBCD_AWS_REGION

# Create a parameter group to set the character set and collation settings:
aws rds create-db-parameter-group \
  --db-parameter-group-name $CBCD_EKS_CLUSTER-mysql-parameters \
  --db-parameter-group-family mysql8.0 \
  --description "MySQL Parameter group for CloudBees CD" \
  --region $CBCD_AWS_REGION

aws rds modify-db-parameter-group \
  --db-parameter-group-name $CBCD_EKS_CLUSTER-mysql-parameters \
  --region $CBCD_AWS_REGION \
  --parameters "ParameterName=character_set_server, ParameterValue=utf8, ApplyMethod=immediate" \
   "ParameterName=character_set_client, ParameterValue=utf8, ApplyMethod=immediate" \
   "ParameterName=character_set_results, ParameterValue=utf8, ApplyMethod=immediate" \
   "ParameterName=collation_server, ParameterValue=utf8_general_ci, ApplyMethod=immediate" \
   "ParameterName=collation_connection, ParameterValue=utf8_general_ci, ApplyMethod=immediate"

# Create database instance (note that this can take 15 mins to complete)
# master username and password must match those you put in the CloudBees CD values.yaml
# file (see later)
aws rds create-db-instance \
  --db-instance-identifier $CBCD_EKS_CLUSTER-db \
  --db-instance-class db.t3.micro \
  --db-name cbcd \
  --db-subnet-group-name cbcd-mysql-subnet \
  --db-parameter-group-name $CBCD_EKS_CLUSTER-mysql-parameters \
  --engine mysql \
  --engine-version 8.0.23 \
  --master-username admin  \
  --master-user-password supersecretpassword \
  --allocated-storage 20 \
  --no-publicly-accessible \
  --region $CBCD_AWS_REGION

# Check database creation status and continue once it is 'available'
aws rds describe-db-instances \
  --db-instance-identifier $CBCD_EKS_CLUSTER-db \
  --region $CBCD_AWS_REGION \
  --query "DBInstances[].DBInstanceStatus"

# Get the endpoint:
CBCD_RDS_Endpoint=$(aws rds describe-db-instances \
  --db-instance-identifier $CBCD_EKS_CLUSTER-db \
  --region $CBCD_AWS_REGION \
  --query "DBInstances[].Endpoint.Address" \
  --output text) && echo $CBCD_RDS_Endpoint

# Get the security group attached to the RDS instance:
CBCD_RDS_SG=$(aws rds describe-db-instances \
  --db-instance-identifier $CBCD_EKS_CLUSTER-db \
  --region $CBCD_AWS_REGION \
  --query "DBInstances[].VpcSecurityGroups[].VpcSecurityGroupId" \
  --output text) && echo $CBCD_RDS_SG

# Accept MySQL traffic:
aws ec2 authorize-security-group-ingress \
  --group-id $CBCD_RDS_SG \
  --cidr $CBCD_CIDR_BLOCK \
  --port 3306 \
  --protocol tcp \
  --region $CBCD_AWS_REGION
```

## Setup ALB ##

```bash
## Associate OIDC provider
eksctl utils associate-iam-oidc-provider \
  --region $CBCD_AWS_REGION \
  --cluster $CBCD_EKS_CLUSTER\
  --approve

## Download the IAM policy document
curl -S https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/v2_ga/docs/install/iam_policy.json -o iam-policy.json

## Create an IAM policy
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json

## Create a service account
eksctl create iamserviceaccount \
  --cluster=$CBCD_EKS_CLUSTER \
  --region $CBCD_AWS_REGION \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --override-existing-serviceaccounts \
  --attach-policy-arn=arn:aws:iam::$CBCD_ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Create a fargate profile so cert-manager can be provisoined on fargate
eksctl create fargateprofile \
  --cluster $CBCD_EKS_CLUSTER \
  --name cert-manager \
  --namespace cert-manager \
  --region $CBCD_AWS_REGION

# Add the AWS chart repo
helm repo add eks https://aws.github.io/eks-charts

# Apply the resources
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"

# Create anew ALB controller
helm install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=$CBCD_EKS_CLUSTER \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set vpcId=$CBCD_VPC_ID \
  --set region=$CBCD_AWS_REGION
```

## Deploy CloudBees CD ##

```bash
# Set environment variables
CBCD_NS=cloudbeescd

# Add a new fargate profile so that Cloudbees CD resources get run on fargate instances.
eksctl create fargateprofile \
  --cluster $CBCD_EKS_CLUSTER \
  --name cloudbeescd \
  --namespace $CBCD_NS \
  --region $CBCD_AWS_REGION

# Update values.yaml for cloudbees CD Helm chart, and set database.externalEndpoint to the value of $CBCD_RDS_Endpoint.  There is an example values.yaml file in the repo (values-example.yaml)
vi values.yaml

# Create namespace
kubectl create ns $CBCD_NS

# Install the cloudbeescd chart:
helm repo add cloudbees https://charts.cloudbees.com/public/cloudbees
helm repo update
helm install cloudbeescd cloudbees/cloudbees-flow -f values.yaml --namespace cloudbeescd --timeout 10000s

# Modify PVC's to name the PV they should bind to:
kubectl patch persistentvolumeclaim/flow-server-shared -n cloudbeescd -p '{"spec": {"volumeName":"flow-server-shared-efs-pv"}}'

kubectl patch persistentvolumeclaim/elasticsearch-data-flow-devopsinsight-0 -n  cloudbeescd -p '{"spec": {"volumeName":"elasticsearch-data-flow-devopsinsight-0-pv"}}'

kubectl patch persistentvolumeclaim/flow-repo-artifacts -n cloudbeescd -p '{"spec": {"volumeName":"flow-repo-artifacts-pv"}}'

# Create PersistentVolumes

echo "
apiVersion: v1
kind: PersistentVolume
metadata:
  name: flow-server-shared-efs-pv
  namespace: cloudbeescd
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: $CBCD_EFS_FS_ID
" | kubectl apply -f -

echo "
apiVersion: v1
kind: PersistentVolume
metadata:
  name: elasticsearch-data-flow-devopsinsight-0-pv
  namespace: cloudbeescd
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: $CBCD_EFS_FS_ID
" | kubectl apply -f -

echo "
apiVersion: v1
kind: PersistentVolume
metadata:
  name: flow-repo-artifacts-pv
  namespace: cloudbeescd
spec:
  capacity:
    storage: 20Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: $CBCD_EFS_FS_ID
" | kubectl apply -f -

# Remove the default ingress
kubectl delete ingress flow-ingress -n cloudbeescd

# Create a new ALB ingress
echo "
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: flow-ingress
  namespace: cloudbeescd
  annotations:
    kubernetes.io/ingress.class: "alb"
    alb.ingress.kubernetes.io/scheme: "internet-facing"
    alb.ingress.kubernetes.io/healthcheck-path: "/"
    alb.ingress.kubernetes.io/success-codes: "200,201,302"
    alb.ingress.kubernetes.io/target-type: "ip"
  labels:
    app: flow
spec:
  rules:
    - http:
        paths:
          - path: /*
            backend:
              serviceName: flow-web
              servicePort: 80
" | kubectl apply -f -
```

You may need to delete the following pods so they get recreated and initialise again:

- flow-web
- flow-server
- flow-repository
- flow-devopsinsight
- flow-bound-agent

Note: CD can take >10 mins to create database, install plugins, and do all the first time setup.

Watch progress with:

```bash
kubectl logs flow-server-XXXX -n cloudbeescd -f
```

At some point, the web page will become activate and you can watch progress there too.
Get the ALB address:

```bash
kubectl get ingress flow-ingress \
  -o jsonpath="{.status.loadBalancer.ingress[].hostname}" -n cloudbeescd
```

Use this address in a web browser and you should get the login screen once the servers have started up.
