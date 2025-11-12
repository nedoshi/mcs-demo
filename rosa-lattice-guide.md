# ROSA HCP with AWS VPC Lattice Setup Guide

## Overview

This guide demonstrates how to connect ROSA HCP applications to databases in another VPC using AWS VPC Lattice, eliminating the need for VPC peering or Transit Gateway.

## Prerequisites

- AWS CLI configured with appropriate credentials
- kubectl installed and configured
- ROSA CLI installed
- Helm 3.x installed
- Two VPCs: one for ROSA HCP cluster, one for RDS database

## Step 1: Set Up Environment Variables

```bash
# Cluster configuration
export CLUSTER_NAME=my-rosa-hcp
export REGION=us-east-1
export PUBLIC_SUBNET_ID=subnet-xxxxx    # Replace with your public subnet
export PRIVATE_SUBNET_ID=subnet-yyyyy   # Replace with your private subnet
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Database configuration
export DB_VPC_ID=vpc-xxxxx              # Replace with your database VPC ID
export RDS_ENDPOINT=mydb.xxxxx.us-east-1.rds.amazonaws.com  # Replace with your RDS endpoint
export RDS_PORT=5432                    # 5432 for PostgreSQL, 3306 for MySQL
```

## Step 2: Create ROSA HCP Cluster

```bash
# Create account roles (if not already created)
rosa create account-roles --mode auto --yes

# Create OIDC configuration
rosa create oidc-config --mode auto --yes

# Get the OIDC ID
export OIDC_ID=$(rosa list oidc-config -o json | jq -r '.[0].id')

# Create the ROSA HCP cluster
rosa create cluster \
  --cluster-name $CLUSTER_NAME \
  --subnet-ids ${PUBLIC_SUBNET_ID},${PRIVATE_SUBNET_ID} \
  --hosted-cp \
  --region $REGION \
  --oidc-config-id $OIDC_ID \
  --sts --mode auto --yes

# Wait for cluster to be ready (this can take 10-15 minutes)
rosa describe cluster --cluster=$CLUSTER_NAME --watch

# Log in to the cluster
rosa create admin --cluster=$CLUSTER_NAME
# Follow the provided login command
```

## Step 3: Get Cluster VPC ID

```bash
# Get your cluster VPC ID
export CLUSTER_VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=*${CLUSTER_NAME}*" \
  --query 'Vpcs[0].VpcId' --output text)

echo "Cluster VPC ID: $CLUSTER_VPC_ID"
```

## Step 4: Install Gateway API CRDs

```bash
# Install Kubernetes Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml

# Verify installation
kubectl get crd | grep gateway
```

## Step 5: Create IAM Policy for VPC Lattice Controller

```bash
# Create IAM policy document
cat > lattice-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "vpc-lattice:*",
        "iam:CreateServiceLinkedRole",
        "ec2:DescribeVpcs",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeTags"
      ],
      "Resource": "*"
    }
  ]
}
EOF

# Create the IAM policy
aws iam create-policy \
  --policy-name VPCLatticeControllerPolicy \
  --policy-document file://lattice-policy.json

# Set policy ARN
export POLICY_ARN=arn:aws:iam::${AWS_ACCOUNT_ID}:policy/VPCLatticeControllerPolicy

echo "Policy ARN: $POLICY_ARN"
```

## Step 6: Install AWS Gateway API Controller

```bash
# Add Helm repository
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Install the Gateway API Controller
helm install gateway-api-controller eks/aws-gateway-controller \
  --namespace aws-application-networking-system \
  --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=${POLICY_ARN} \
  --set clusterVpcId=${CLUSTER_VPC_ID} \
  --set clusterName=${CLUSTER_NAME} \
  --set region=${REGION}

# Verify installation
kubectl get pods -n aws-application-networking-system
kubectl get deployment -n aws-application-networking-system
```

## Step 7: Create VPC Lattice Service Network

```bash
# Create a service network
aws vpc-lattice create-service-network \
  --name my-service-network \
  --region $REGION

# Get the service network ID
export SN_ID=$(aws vpc-lattice list-service-networks \
  --query "items[?name=='my-service-network'].id" \
  --output text)

echo "Service Network ID: $SN_ID"

# Associate ROSA cluster VPC with the service network
aws vpc-lattice create-service-network-vpc-association \
  --service-network-identifier $SN_ID \
  --vpc-identifier $CLUSTER_VPC_ID \
  --region $REGION

# Associate database VPC with the service network
aws vpc-lattice create-service-network-vpc-association \
  --service-network-identifier $SN_ID \
  --vpc-identifier $DB_VPC_ID \
  --region $REGION

# Verify associations
aws vpc-lattice list-service-network-vpc-associations \
  --service-network-identifier $SN_ID \
  --region $REGION
```

## Step 8: Create VPC Lattice Target Group for RDS

```bash
# Create target group for RDS database
aws vpc-lattice create-target-group \
  --name my-rds-targets \
  --type IP \
  --config protocol=TCP,port=$RDS_PORT,vpcIdentifier=$DB_VPC_ID \
  --region $REGION

# Get target group ID
export TG_ID=$(aws vpc-lattice list-target-groups \
  --query "items[?name=='my-rds-targets'].id" \
  --output text)

echo "Target Group ID: $TG_ID"

# Get RDS instance private IP (if using IP targets)
export RDS_IP=$(dig +short $RDS_ENDPOINT | head -n1)
echo "RDS IP: $RDS_IP"

# Register RDS as a target
aws vpc-lattice register-targets \
  --target-group-identifier $TG_ID \
  --targets id=$RDS_IP,port=$RDS_PORT \
  --region $REGION

# Verify target registration
aws vpc-lattice list-targets \
  --target-group-identifier $TG_ID \
  --region $REGION
```

## Step 9: Create VPC Lattice Service

```bash
# Create a VPC Lattice service
aws vpc-lattice create-service \
  --name my-rds-service \
  --region $REGION

# Get service ID
export SERVICE_ID=$(aws vpc-lattice list-services \
  --query "items[?name=='my-rds-service'].id" \
  --output text)

echo "Service ID: $SERVICE_ID"

# Associate service with service network
aws vpc-lattice create-service-network-service-association \
  --service-network-identifier $SN_ID \
  --service-identifier $SERVICE_ID \
  --region $REGION

# Create a listener for the service
aws vpc-lattice create-listener \
  --service-identifier $SERVICE_ID \
  --name tcp-listener \
  --protocol TCP \
  --port $RDS_PORT \
  --default-action "{\"forward\":{\"targetGroups\":[{\"targetGroupIdentifier\":\"$TG_ID\"}]}}" \
  --region $REGION

# Get the service DNS name
export SERVICE_DNS=$(aws vpc-lattice get-service \
  --service-identifier $SERVICE_ID \
  --query 'dnsEntry.domainName' \
  --output text)

echo "Service DNS: $SERVICE_DNS"
```

## Step 10: Update RDS Security Group

```bash
# Get the VPC Lattice managed prefix list ID
export LATTICE_PREFIX_LIST=$(aws ec2 describe-managed-prefix-lists \
  --filters "Name=prefix-list-name,Values=com.amazonaws.$REGION.vpc-lattice" \
  --query 'PrefixLists[0].PrefixListId' \
  --output text)

echo "VPC Lattice Prefix List: $LATTICE_PREFIX_LIST"

# Get RDS security group ID
export RDS_SG=$(aws rds describe-db-instances \
  --query "DBInstances[?Endpoint.Address=='$RDS_ENDPOINT'].VpcSecurityGroups[0].VpcSecurityGroupId" \
  --output text)

echo "RDS Security Group: $RDS_SG"

# Add inbound rule to RDS security group
aws ec2 authorize-security-group-ingress \
  --group-id $RDS_SG \
  --ip-permissions IpProtocol=tcp,FromPort=$RDS_PORT,ToPort=$RDS_PORT,PrefixListIds=[{PrefixListId=$LATTICE_PREFIX_LIST}] \
  --region $REGION
```

## Step 11: Create Kubernetes Service

```bash
# Create a Kubernetes service that points to VPC Lattice
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: rds-database
  namespace: default
spec:
  type: ExternalName
  externalName: $SERVICE_DNS
  ports:
  - port: $RDS_PORT
    protocol: TCP
    targetPort: $RDS_PORT
EOF

# Verify service creation
kubectl get svc rds-database
```

## Step 12: Create Gateway Configuration

```bash
# Create Gateway resource
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: my-lattice-gateway
  namespace: default
spec:
  gatewayClassName: amazon-vpc-lattice
  listeners:
  - name: tcp-listener
    protocol: TCP
    port: $RDS_PORT
EOF

# Verify gateway creation
kubectl get gateway my-lattice-gateway
kubectl describe gateway my-lattice-gateway
```

## Step 13: Test Database Connectivity

```bash
# For PostgreSQL
kubectl run test-postgres --image=postgres:latest -it --rm --restart=Never -- \
  psql -h rds-database -p $RDS_PORT -U your_username -d your_database

# For MySQL
kubectl run test-mysql --image=mysql:latest -it --rm --restart=Never -- \
  mysql -h rds-database -P $RDS_PORT -u your_username -p your_database

# Alternative: Deploy a test pod and exec into it
kubectl run test-pod --image=busybox --restart=Never --command -- sleep 3600

# Test DNS resolution
kubectl exec test-pod -- nslookup rds-database

# Test TCP connectivity (replace with your port)
kubectl exec test-pod -- nc -zv rds-database $RDS_PORT

# Clean up test pod
kubectl delete pod test-pod
```

## Step 14: Deploy Sample Application

```bash
# Create a sample application that connects to the database
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-client-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db-client
  template:
    metadata:
      labels:
        app: db-client
    spec:
      containers:
      - name: app
        image: postgres:latest  # or mysql:latest
        command: ["sleep", "infinity"]
        env:
        - name: DB_HOST
          value: "rds-database"
        - name: DB_PORT
          value: "$RDS_PORT"
        - name: DB_NAME
          value: "your_database"
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
EOF

# Create secret for database credentials (replace with your values)
kubectl create secret generic db-credentials \
  --from-literal=username=your_username \
  --from-literal=password=your_password

# Verify deployment
kubectl get deployment db-client-app
kubectl get pods -l app=db-client
```

## Step 15: Configure Auth Policies (Optional)

```bash
# Create an auth policy for the service network
cat > auth-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::${AWS_ACCOUNT_ID}:root"
      },
      "Action": "vpc-lattice-svcs:Invoke",
      "Resource": "*"
    }
  ]
}
EOF

# Apply auth policy to service network
aws vpc-lattice put-auth-policy \
  --resource-identifier $SN_ID \
  --policy file://auth-policy.json \
  --region $REGION

# Verify auth policy
aws vpc-lattice get-auth-policy \
  --resource-identifier $SN_ID \
  --region $REGION
```

## Monitoring and Troubleshooting

```bash
# Check VPC Lattice service status
aws vpc-lattice get-service \
  --service-identifier $SERVICE_ID \
  --region $REGION

# Check target health
aws vpc-lattice list-targets \
  --target-group-identifier $TG_ID \
  --region $REGION

# View Gateway API controller logs
kubectl logs -n aws-application-networking-system \
  -l app.kubernetes.io/name=aws-gateway-controller \
  --tail=100 -f

# Check service network associations
aws vpc-lattice list-service-network-vpc-associations \
  --service-network-identifier $SN_ID \
  --region $REGION

# Enable VPC Lattice access logs (optional)
aws vpc-lattice update-service \
  --service-identifier $SERVICE_ID \
  --access-log-settings destinationArn=arn:aws:s3:::your-log-bucket \
  --region $REGION
```

## Cleanup

```bash
# Delete Kubernetes resources
kubectl delete deployment db-client-app
kubectl delete secret db-credentials
kubectl delete service rds-database
kubectl delete gateway my-lattice-gateway

# Delete VPC Lattice resources
aws vpc-lattice delete-listener \
  --service-identifier $SERVICE_ID \
  --listener-identifier <listener-id> \
  --region $REGION

aws vpc-lattice delete-service-network-service-association \
  --service-network-service-association-identifier <association-id> \
  --region $REGION

aws vpc-lattice deregister-targets \
  --target-group-identifier $TG_ID \
  --targets id=$RDS_IP \
  --region $REGION

aws vpc-lattice delete-target-group \
  --target-group-identifier $TG_ID \
  --region $REGION

aws vpc-lattice delete-service \
  --service-identifier $SERVICE_ID \
  --region $REGION

aws vpc-lattice delete-service-network-vpc-association \
  --service-network-vpc-association-identifier <association-id> \
  --region $REGION

aws vpc-lattice delete-service-network \
  --service-network-identifier $SN_ID \
  --region $REGION

# Uninstall Gateway API controller
helm uninstall gateway-api-controller \
  -n aws-application-networking-system

# Delete ROSA cluster
rosa delete cluster --cluster=$CLUSTER_NAME --yes
```

## Key Benefits

- **No VPC Peering Required**: Direct service-to-service connectivity across VPCs
- **No Transit Gateway Costs**: Eliminate complex networking infrastructure
- **Handles Overlapping CIDRs**: Works even with overlapping IP ranges
- **Fine-grained Security**: IAM-based access control for services
- **Automatic Service Discovery**: DNS-based service resolution
- **Regional Service**: All resources must be in the same AWS region

## Additional Resources

- [AWS VPC Lattice Documentation](https://docs.aws.amazon.com/vpc-lattice/)
- [ROSA Documentation](https://docs.openshift.com/rosa/)
- [AWS Gateway API Controller](https://github.com/aws/aws-application-networking-k8s)
- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)