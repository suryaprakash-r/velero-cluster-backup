# Velero Backup & Restore on AWS EKS

## What is Velero?
Velero is an open-source tool designed for **backing up, restoring, and migrating Kubernetes clusters and persistent volumes**. It helps ensure **disaster recovery, data protection, and cluster portability** across cloud environments.

### Why Use Velero?
✅ **Disaster Recovery** – Quickly restore your Kubernetes workloads in case of failure.  
✅ **Cluster Migration** – Move applications across clusters or cloud providers seamlessly.  
✅ **Scheduled Backups** – Automate regular backups for better data resilience.  
✅ **Persistent Volume Snapshots** – Ensure your storage remains protected.  

With Velero, you can **safeguard your Kubernetes workloads** while keeping backup and restore processes efficient and scalable. 🚀

---

## Velero Backup & Restore on AWS EKS

### Step 1: Install AWS CLI & Configure IAM User
```sh
aws configure
aws sts get-caller-identity
```

### Step 2: Create an S3 Bucket for Velero Backups
```sh
aws s3api create-bucket --bucket eks-velero-backup-devops --region ap-south-1 --create-bucket-configuration LocationConstraint=ap-south-1

# Enable versioning for the bucket
aws s3api put-bucket-versioning --bucket eks-velero-backup-devops --versioning-configuration Status=Enabled
```

### Step 3: Create an IAM Role for Velero
#### Create a `velero-policy.json` file for IAM permissions:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListAllMyBuckets"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::eks-velero-backup-devops",
                "arn:aws:s3:::eks-velero-backup-devops/*"
            ]
        }
    ]
}
```

#### Create an IAM Policy:
```sh
aws iam create-policy --policy-name VeleroBackupPolicyEKS --policy-document file://velero-policy.json
```

#### Create an IAM Role for Velero:
```sh
eksctl create iamserviceaccount \
  --cluster blue-green-cluster \
  --namespace velero \
  --name velero \
  --attach-policy-arn arn:aws:iam::<account-id>:policy/VeleroBackupPolicyEKS \
  --approve
```

#### Verify the Service Account:
```sh
kubectl get sa -n velero
```

### Step 4: Install Velero on EKS
#### Install Velero CLI (if not already installed):
```sh
curl -LO https://github.com/vmware-tanzu/velero/releases/latest/download/velero-linux-amd64.tar.gz
tar -xvf velero-linux-amd64.tar.gz
sudo mv velero-linux-amd64/velero /usr/local/bin/
```

#### Install Velero:
```sh
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.0.1 \
  --bucket eks-velero-backup-devops \
  --backup-location-config region=ap-south-1 \
  --snapshot-location-config region=ap-south-1 \
  --secret-file credentials-velero
```

#### Verify Velero Deployment:
```sh
kubectl get all -n velero
kubectl get volumesnapshotlocations -n velero
kubectl get backupstoragelocations -n velero
```

### Step 5: Create & Manage Backups
#### Create a Backup:
```sh
velero backup create eks-devops-cluster --include-namespaces webapps
```
#### Check Backup Status:
```sh
velero backup get
velero backup describe eks-devops-cluster
```
#### Schedule Daily Backups:
```sh
velero schedule create daily-backup --schedule="0 0 * * *" --include-namespaces default
```

---
