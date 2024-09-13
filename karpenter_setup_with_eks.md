
# Karpenter Configuration with AWS EKS Cluster

## Prerequisites

Before configuring Karpenter, you need:
1. **AWS CLI** installed and configured with necessary permissions.
2. **kubectl** installed to interact with your Kubernetes cluster.
3. **Helm** installed (optional but recommended).
4. **IAM Permissions**: Ensure your user has permissions to create IAM roles, policies, EC2 instances, etc.
5. **An existing EKS cluster** (running Kubernetes 1.18+).

## Step 1: Create an EKS Cluster
If you don't already have an EKS cluster, follow these steps:
```bash
eksctl create cluster \
  --name karpenter-eks-cluster \
  --region <region> \
  --nodes 2 \
  --node-type t3.medium
```

## Step 2: Install AWS Load Balancer Controller (Optional)
```bash
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=karpenter-eks-cluster \
  --set serviceAccount.create=false \
  --set region=<region>
```

## Step 3: Create IAM Role for Karpenter

### 3.1 Create a Karpenter Controller Role:
1. Download the required IAM policy for Karpenter:
   ```bash
   curl -o karpenter-controller-policy.json https://karpenter.sh/v0.5.0/karpenter-controller-policy.json
   ```
2. Create the IAM policy:
   ```bash
   aws iam create-policy --policy-name KarpenterControllerPolicy --policy-document file://karpenter-controller-policy.json
   ```
3. Create a service-linked role for Karpenter:
   ```bash
   eksctl create iamserviceaccount \
     --cluster karpenter-eks-cluster \
     --namespace karpenter \
     --name karpenter \
     --attach-policy-arn arn:aws:iam::<account-id>:policy/KarpenterControllerPolicy \
     --region <region> \
     --approve
   ```

## Step 4: Deploy Karpenter in the EKS Cluster

### 4.1 Helm Installation (Recommended)
1. Add the Karpenter Helm repo:
   ```bash
   helm repo add karpenter https://charts.karpenter.sh
   helm repo update
   ```
2. Install Karpenter using Helm:
   ```bash
   helm upgrade --install karpenter karpenter/karpenter \
     --namespace karpenter \
     --create-namespace \
     --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::<account-id>:role/<role-name> \
     --set settings.aws.clusterName=karpenter-eks-cluster \
     --set settings.aws.defaultInstanceProfile=KarpenterInstanceProfile \
     --set settings.aws.interruptionQueueName=karpenter-interruption-queue \
     --set settings.aws.clusterEndpoint=<cluster-endpoint>
   ```

### 4.2 Manifest Installation
Alternatively, you can deploy Karpenter using raw Kubernetes manifests:
```bash
kubectl apply -f https://karpenter.sh/v0.5.0/karpenter.yaml
```

## Step 5: Create Instance Profile for Karpenter

Karpenter launches EC2 instances, so it requires an EC2 instance profile to attach to those nodes.

1. Create an EC2 instance profile:
   ```bash
   aws iam create-instance-profile --instance-profile-name KarpenterInstanceProfile
   ```
2. Attach the Karpenter role to the instance profile:
   ```bash
   aws iam add-role-to-instance-profile --instance-profile-name KarpenterInstanceProfile --role-name <karpenter-node-role>
   ```

## Step 6: Create a Provisioner
Karpenter uses provisioners to launch and scale nodes based on your cluster needs.

1. Create a provisioner YAML file:
```yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  cluster:
    name: karpenter-eks-cluster
  requirements:
    - key: "node.kubernetes.io/instance-type"
      operator: In
      values: ["m5.large", "m5.xlarge"]
  provider:
    instanceProfile: "KarpenterInstanceProfile"
  limits:
    resources:
      cpu: "1000"
  ttlSecondsAfterEmpty: 30
```

2. Apply the provisioner to the cluster:
```bash
kubectl apply -f provisioner.yaml
```

## Step 7: Configure Karpenter for Spot Instances (Optional)

Karpenter supports using EC2 Spot Instances for cost-saving.

1. Add Spot instance types to the provisioner config:
```yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: spot-instance-provisioner
spec:
  cluster:
    name: karpenter-eks-cluster
  requirements:
    - key: "node.kubernetes.io/instance-type"
      operator: In
      values: ["m5.large", "m5.xlarge"]
    - key: "karpenter.k8s.aws/capacity-type"
      operator: In
      values: ["spot"]
  provider:
    instanceProfile: "KarpenterInstanceProfile"
  limits:
    resources:
      cpu: "500"
  ttlSecondsAfterEmpty: 60
```

2. Apply the updated provisioner:
```bash
kubectl apply -f spot-instance-provisioner.yaml
```

## Step 8: Monitor and Test
Test that Karpenter scales your nodes by scheduling some workloads:
1. Deploy a workload:
   ```bash
   kubectl run nginx --image=nginx --replicas=10
   ```
2. Watch Karpenter in action:
   ```bash
   kubectl get pods -o wide
   ```

## Step 9: Enable Monitoring and Logging (Optional)

To gain insights into Karpenter’s behavior, enable logging and metrics:
1. View logs:
   ```bash
   kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter
   ```
2. Use **Prometheus** or **CloudWatch** to track node scaling events and metrics.

## Conclusion
You’ve now successfully configured Karpenter with your AWS EKS cluster! Karpenter will dynamically provision nodes based on your application’s needs.
