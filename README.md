# helm-thanos

Example Helm chart for setting up Thanos in your Kubernetes cluster.

Chart Reference - https://github.com/banzaicloud/banzai-charts/tree/master/thanos

Table of Contents
=================
   * [helm-thanos](#helm-thanos)
   * [Table of Contents](#table-of-contents)
      * [Pre-requisites](#pre-requisites)
         * [Namespace](#namespace)
      * [AWS](#aws)
         * [IAM](#iam)
            * [Service Account](#service-account)
         * [Config Updates](#config-updates)
      * [Install/Upgrade Chart](#installupgrade-chart)

## Pre-requisites

### Namespace

Create a new namespace `platform` where we will install thanos.

```bash
kubectl create namespace platform
```

## AWS

### IAM

We need to give permissions for Thanos store and compact to manage the metric files in S3.

We will be using IRSA (IAM Roles for Service Accounts) to give the required permissions.

`Note: You need to create an OIDC provider for your cluster to make use of IRSA. Refer - https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html`

1. Create an IAM policy named `k8s-thanos-store-pol` with below policy document.

Replace `bucket_name` with the bucket name of your metric files.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3Operations",
      "Action": [
        "s3:ListBucketMultipartUploads",
        "s3:ListMultipartUploadParts",
        "s3:ListBucket",
        "s3:GetObject",
        "s3:GetObjectTagging",
        "s3:PutObjectTagging",
        "s3:AbortMultipartUpload",
        "s3:DeleteObject",
        "s3:PutObject"
      ],
      "Effect": "Allow",
      "Resource": [
        "arn:aws:s3:::${bucket_name}/*",
        "arn:aws:s3:::${bucket_name}"
      ]
    }
  ]
} 
```

2. Create an IAM role `k8s-thanos-store-rol`. Attach the IAM policies which we had created earlier.

3. Update the trust relationship of the IAM roles as below replacing the `account_id`, `eks_cluster_id` and `region` with the appropriate values.

For example, this trust relationship allows pods with serviceaccount `thanos-store` in `platform` namespace to assume the role.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<account_id>:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/<eks_cluster_id>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.<region>.amazonaws.com/id/<eks_cluster_id>:sub": "system:serviceaccount:platform:thanos-store"
        }
      }
    }
  ]
}
```

#### Service Account

Create new service accounts in the `platform` namespace and associate it with the IAM roles which we had created earlier.

e.g.

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: thanos-store
  namespace: platform
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/k8s-thanos-store-rol
EOF
```

We already specified the service account to be used by the pods in the config file `stages/aws-shared-values.yaml`.

### Config Updates

In `aws-prod-values.yaml` file available inside `stages/prod` folder, add values for below settings:

|||
|--|--|
|bucket |Metrics bucket name |
|stores |By default, it has DNS SRV record to the envoy proxy running in the same cluster. You can additionally provide DNS names to Thanos sidecar instances running in other clusters. |

## Install/Upgrade Chart

Sample command to install/upgrade thanos.

```bash
helm upgrade -i thanos . -n platform --values stages/aws-shared-values.yaml --values stages/prod/aws-prod-values.yaml
```
