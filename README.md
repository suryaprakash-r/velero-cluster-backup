**Setting Up Velero for EKS Cluster Backup**

## Step 1: Install AWS CLI & Configure IAM User

Configure AWS credentials:
```sh
aws configure
```
Verify IAM user identity:
```sh
aws sts get-caller-identity
```

## Step 2: Create an S3 Bucket for Velero Backups

Create an S3 bucket:
```sh
aws s3api create-bucket --bucket eks-velero-backup-devops --region ap-south-1 --create-bucket-configuration LocationConstraint=ap-south-1
```

Enable versioning for the bucket:
```sh
aws s3api put-bucket-versioning --bucket eks-velero-backup-devops --versioning-configuration Status=Enabled
```

## Step 3: Create an IAM Role for Velero

### Create a Velero Policy (velero-policy.json)
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

Create an IAM policy:
```sh
aws iam create-policy --policy-name VeleroBackupPolicyEKS --policy-document file://velero-policy.json
```

Create an IAM service account for Velero:
```sh
eksctl create iamserviceaccount \
  --cluster blue-green-cluster \
  --namespace velero \
  --name velero \
  --attach-policy-arn arn:aws:iam::<account-id>:policy/VeleroBackupPolicyEKS \
  --approve
```

Verify the service account:
```sh
kubectl get sa -n velero
```

## Step 4: Install Velero on EKS

### Install Velero CLI (if not already installed)
```sh
curl -LO https://github.com/vmware-tanzu/velero/releases/latest/download/velero-linux-amd64.tar.gz
tar -xvf velero-linux-amd64.tar.gz
sudo mv velero-linux-amd64/velero /usr/local/bin/
```

### Install Velero on EKS
```sh
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.0.1 \
  --bucket eks-velero-backup-devops \
  --backup-location-config region=ap-south-1 \
  --snapshot-location-config region=ap-south-1 \
  --secret-file credentials-velero
```

### Verify Installation
```sh
kubectl get all -n velero
kubectl get volumesnapshotlocations -n velero
kubectl get backupstoragelocations -n velero
```

## Step 5: Create a Backup

Create a backup for the `webapps` namespace:
```sh
velero backup create eks-devops-cluster --include-namespaces webapps
```

List backups:
```sh
velero backup get
```

Describe a specific backup:
```sh
velero backup describe eks-devops-cluster
```

## Step 6: Automate Scheduled Backups

Create a daily backup schedule:
```sh
velero schedule create daily-backup --schedule="0 0 * * *" --include-namespaces default
```

