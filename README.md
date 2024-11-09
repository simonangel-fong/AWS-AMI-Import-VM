# AWS AMI: Import VM Image

- A repo of migrating an on-premises VM to AWS AMI.

  - OS images are created and export using VMware Workstation locally, and then upload to S3 and register in AMI.

- [AWS AMI: Import VM Image](#aws-ami-import-vm-image)
  - [Create the service role](#create-the-service-role)
  - [Create policy for S3 bucket](#create-policy-for-s3-bucket)
  - [Upload `.ova` to S3 bucket](#upload-ova-to-s3-bucket)
  - [Import `.ova` from S3 to AMI](#import-ova-from-s3-to-ami)

---

## Create the service role

- Create a file named `trust-policy.json`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "vmie.amazonaws.com" },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:Externalid": "vmimport"
        }
      }
    }
  ]
}
```

- Use the create-role command to create a role named vmimport and grant VM Import/Export access to it.

```sh
aws iam create-role --role-name vmimport --assume-role-policy-document "file://path\to\trust-policy.json"
```

---

## Create policy for S3 bucket

- Create a file named `role-policy.json` with the following policy.
  - `amzn-s3-demo-import-bucket`: the bucket for imported disk images
  - `amzn-s3-demo-export-bucket`: the bucket for exported disk images

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetBucketLocation", "s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::amzn-s3-demo-import-bucket",
        "arn:aws:s3:::amzn-s3-demo-import-bucket/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation",
        "s3:GetObject",
        "s3:ListBucket",
        "s3:PutObject",
        "s3:GetBucketAcl"
      ],
      "Resource": [
        "arn:aws:s3:::amzn-s3-demo-export-bucket",
        "arn:aws:s3:::amzn-s3-demo-export-bucket/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:ModifySnapshotAttribute",
        "ec2:CopySnapshot",
        "ec2:RegisterImage",
        "ec2:Describe*"
      ],
      "Resource": "*"
    }
  ]
}
```

- Use the following put-role-policy command to attach the policy to the role created above.

```sh
aws iam put-role-policy --role-name vmimport --policy-name vmimport --policy-document "file://path\to\role-policy.json"
```

---

## Upload `.ova` to S3 bucket

```sh
aws s3 cp OL8-General.ova s3://simonangel-ec2-ami
```

---

## Import `.ova` from S3 to AMI

- Create `containers.json` file that specifies the image using an S3 bucket.

```json
[
  {
    "Description": "My Server OVA",
    "Format": "ova",
    "Url": "s3://amzn-s3-demo-import-bucket/vms/my-server-vm.ova"
  }
]
```

- command to import an image with a single disk.

```sh
aws ec2 import-image --description "My server VM" --disk-containers "file://path\to\containers.json"
```

- Monitor an import image task

```sh
aws ec2 describe-import-image-tasks --import-task-ids import-ami-number
```
