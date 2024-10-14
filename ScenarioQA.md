# Q. Suppose you have an application deployed inside EKS, and your application needs some secrets to run. These secrets are stored in the Secrets Manager service in AWS. How will you provision this so that your application can access those secrets and configurations?

To provision your application deployed inside Amazon EKS to access secrets from AWS Secrets Manager, you can follow these steps:
1. **IAM Role for Service Account (IRSA) Setup**
The best practice for allowing EKS pods to access AWS services, such as Secrets Manager, is to use IAM Role for Service Account (IRSA). This avoids the need to inject AWS credentials directly into your pods and instead leverages fine-grained IAM roles associated with Kubernetes service accounts.

### Note - An `OIDC Provider` (OP) is a server that authenticates users and issues identity tokens in an OpenID Connect `(OIDC)` flow, enabling secure user identity verification for client applications. It builds on OAuth 2.0 to facilitate `Single Sign-On` (SSO) and secure access across multiple services.
**Step-by-Step Process**:
- **Step 1**: Enable OIDC Provider for Your EKS Cluster
First, ensure that your EKS cluster has an OIDC provider enabled, which is necessary for associating Kubernetes service accounts with IAM roles.
```bash
# Get the OIDC provider URL for your cluster
aws eks describe-cluster --name <your-cluster-name> --query "cluster.identity.oidc.issuer" --output text

# Create the OIDC identity provider if not already done
eksctl utils associate-iam-oidc-provider --cluster <your-cluster-name> --approve
```
- **Step 2**: Create an IAM Policy for Secrets Manager Access
You need to create an IAM policy that grants access to Secrets Manager for your application. This policy will be attached to the IAM role that your pod will assume.
```bash
aws iam create-policy --policy-name EKSSecretsManagerPolicy --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue"
            ],
            "Resource": [
                "arn:aws:secretsmanager:<region>:<account-id>:secret:<your-secret-name>"
            ]
        }
    ]
}'
```
- **Step 3**: Create an IAM Role and Associate it with a Service Account
Create an IAM role that allows access to the Secrets Manager using the policy you created, and associate it with a Kubernetes service account. Pods using this service account will assume this role.
```bash
eksctl create iamserviceaccount \
  --name my-app-sa \
  --namespace default \
  --cluster <your-cluster-name> \
  --attach-policy-arn arn:aws:iam::<account-id>:policy/EKSSecretsManagerPolicy \
  --approve \
  --override-existing-serviceaccounts
```
This command creates a Kubernetes service account `my-app-sa` in the `default` namespace, attaches the `EKSSecretsManagerPolicy`, and associates it with the IAM 
role.
- **Step 4**: Deploy the Application with the Service Account
Update your application deployment to use the service account you just created. Modify your Kubernetes Deployment YAML file to include the service account name.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: my-app-sa
      containers:
      - name: my-app
        image: <your-application-image>
        env:
        - name: SECRET_VALUE
          valueFrom:
            secretKeyRef:
              name: my-app-secret
              key: secret-key
```
- **Step 5**: Access the Secret from the Application
In your application code, use the AWS SDK to access the secret from Secrets Manager. Here's an example of Python code to retrieve a secret:
```python 
import boto3
import os

def get_secret():
    secret_name = os.getenv("SECRET_NAME")
    region_name = os.getenv("AWS_REGION")

    # Create a Secrets Manager client
    session = boto3.session.Session()
    client = session.client(
        service_name='secretsmanager',
        region_name=region_name
    )

    get_secret_value_response = client.get_secret_value(
        SecretId=secret_name
    )

    return get_secret_value_response['SecretString']
```
### Summary
- IAM Role for Service Account (IRSA): Leverage IRSA to securely access Secrets Manager from your EKS pods.
- Secrets Manager IAM Policy: Create an IAM policy allowing your pods to access specific secrets.
- Service Account with IAM Role: Create a service account in Kubernetes that is associated with the IAM role.
- Application Access: Use the AWS SDK within your application to fetch secrets from Secrets Manager.

# Q. Suppose you have a database that needs to be deployed on Kubernetes, and it needs to be highly available. How would you achieve that? 
To deploy a highly available (HA) database on Kubernetes, you need to ensure redundancy, failover capability, and data replication, so that the database remains operational even in the event of node or pod failures. Below is a step-by-step approach to achieve high availability for a database in Kubernetes:

### 1. Use StatefulSets for Database Deployment
**StatefulSets** are essential for deploying stateful applications like databases on Kubernetes because they provide stable network identifiers, stable storage, and ordered deployment and scaling. StatefulSets maintain the state across pod restarts.
Here’s an example of deploying a PostgreSQL StatefulSet:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```
### 2. Configure Persistent Storage (PersistentVolumes)
Since databases require persistent storage to maintain data across pod restarts, you’ll need to configure PersistentVolumeClaims `(PVCs)` for each pod. The storage must be highly available, either via a distributed storage solution or cloud-managed storage classes (e.g., AWS EBS, Google Persistent Disks).
For high availability:
- Use a storage class that replicates data across different availability zones.
- For on-prem or self-managed clusters, use distributed storage like Ceph, Rook, or Portworx.
```yaml
volumeClaimTemplates:
- metadata:
    name: postgres-data
  spec:
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 10Gi
    storageClassName: "your-ha-storage-class"
```
### 3. **Replication and Failover within the Database**
To achieve high availability, set up replication within your database. Depending on the database technology, you can use one of the following:
- **PostgreSQL**: Set up Streaming Replication or use tools like Patroni to handle leader election and automatic failover.
- **MySQL**: Use MySQL Group Replication or MySQL InnoDB Cluster.
- **MongoDB**: Deploy a Replica Set to ensure redundancy and automatic failover.
- **Redis**: Use Redis Sentinel for HA or Redis Cluster for distributed deployment.
For PostgreSQL, a solution like Patroni is popular for managing high availability:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: patroni
spec:
  serviceName: patroni
  replicas: 3
  selector:
    matchLabels:
      app: patroni
  template:
    metadata:
      labels:
        app: patroni
    spec:
      containers:
      - name: patroni
        image: zalando/patroni:latest
        ports:
        - containerPort: 5432
        envFrom:
        - configMapRef:
            name: patroni-config
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```
### 4. Service Configuration for Failover and Load Balancing
Use Kubernetes Services to expose your database to applications and handle failover. There are two main types of services you can use:

- **Headless Services**: Allow direct pod-to-pod communication (important for master-slave or master-replica configurations). Use `clusterIP: None`for a headless service.
- **Separate Services for Read and Write Operations**:
   - One service for write operations, pointing to the master (primary) database.
   - Another service for read operations, pointing to read replicas.
Example of a headless service for a StatefulSet:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  ports:
  - port: 5432
    name: postgres
  clusterIP: None  # Headless service for StatefulSet
  selector:
    app: postgres
```
### 5. Pod Anti-Affinity and Zone Distribution
Ensure that your database pods are distributed across multiple nodes and availability zones (if applicable) using Pod Anti-Affinity rules. This prevents all replicas from being placed on the same node, which would create a single point of failure.
```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - postgres
      topologyKey: "kubernetes.io/hostname"
```
This rule ensures that pods with the label `app: postgres` are scheduled on different nodes.
### 6. Monitoring and Health Checks
Ensure that the health of your database is being monitored and that Kubernetes can automatically restart unhealthy pods:
  - **Liveness and Readiness Probes**: Configure health checks for your database pods to detect if they’re unhealthy and should be restarted.
For example, a readiness probe for PostgreSQL:
```yaml
readinessProbe:
  exec:
    command:
    - pg_isready
  initialDelaySeconds: 10
  periodSeconds: 5
```
 - Use monitoring tools like Prometheus and Grafana to track metrics such as query performance, replication lag, disk usage, and pod health.
 - Set up alerting to notify you of issues like high CPU usage, replication lag, or failed health checks.

### 7. Automated Backup and Restore
Ensure you have a robust backup and restore strategy. You can use Kubernetes CronJobs to schedule regular backups of your database.
Here’s an example of a CronJob that backs up PostgreSQL to an S3 bucket:
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
spec:
  schedule: "0 0 * * *"  # Run daily at midnight
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: postgres-backup
            image: postgres:13
            command:
            - /bin/sh
            - -c
            - "pg_dump -U postgres dbname | gzip > /backup/dbname.gz && aws s3 cp /backup/dbname.gz s3://your-backup-bucket/"
            env:
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: access_key_id
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: secret_access_key
          restartPolicy: OnFailure
```
### 8. Disaster Recovery
For disaster recovery, ensure that your backup data is stored in a separate region or availability zone. In the event of a failure, you should have a process for restoring data from backups.

## Summary of Key Steps
