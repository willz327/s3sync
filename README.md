# S3 Cross-Partition Sync Solution

A secure, serverless solution for automatically synchronizing S3 objects between AWS China regions and AWS Global regions using Lambda, EventBridge, and Secrets Manager.

## ✨ Features

- 🌏 **Cross-Partition Support**: Sync from AWS China (cn-northwest-1, cn-north-1) to any AWS Global region
- 🔐 **Secure Credential Management**: Uses AWS Secrets Manager to store target account credentials
- ⚡ **Event-Driven**: Automatically triggered by S3 object creation via EventBridge
- 🚀 **Intelligent Sync**: ETag-based incremental synchronization to avoid unnecessary transfers
- 📦 **Multi-Size File Support**: Optimized handling for both small files (memory) and large files (/tmp)
- 🔄 **Cross-Account**: Supports synchronization between different AWS accounts
- 📊 **Configurable**: Customizable Lambda memory, timeout, and tmp directory size
- 🛡️ **Production Ready**: Comprehensive error handling and logging

## 🏗️ Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   AWS China     │    │   EventBridge    │    │  AWS Global     │
│                 │    │                  │    │                 │
│  ┌───────────┐  │    │  ┌─────────────┐ │    │  ┌───────────┐  │
│  │ Source S3 │──┼────┼──│ Event Rule  │ │    │  │ Target S3 │  │
│  │  Bucket   │  │    │  └─────────────┘ │    │  │  Bucket   │  │
│  └───────────┘  │    │         │        │    │  └───────────┘  │
│                 │    │         ▼        │    │         ▲       │
│  ┌───────────┐  │    │  ┌─────────────┐ │    │         │       │
│  │  Lambda   │◄─┼────┼──│   Trigger   │ │    │         │       │
│  │ Function  │  │    │  └─────────────┘ │    │         │       │
│  └───────────┘  │    └──────────────────┘    │         │       │
│         │       │                            │         │       │
│         ▼       │                            │         │       │
│  ┌───────────┐  │                            │         │       │
│  │ Secrets   │  │                            │         │       │
│  │ Manager   │  │                            │         │       │
│  └───────────┘  │                            │         │       │
│                 │    ┌──────────────────────────────────┘       │
└─────────────────┘    │         Cross-Partition Sync            │
                       └──────────────────────────────────────────┘
```

## 🌍 Supported Regions

### Source Regions (AWS China)
- `cn-northwest-1` (Ningxia)
- `cn-north-1` (Beijing)

### Target Regions (AWS Global)
| Region | Location | Region | Location |
|--------|----------|--------|----------|
| `us-east-1` | N. Virginia | `eu-west-1` | Ireland |
| `us-east-2` | Ohio | `eu-west-2` | London |
| `us-west-1` | N. California | `eu-west-3` | Paris |
| `us-west-2` | Oregon | `eu-central-1` | Frankfurt |
| `ap-southeast-1` | Singapore | `eu-north-1` | Stockholm |
| `ap-southeast-2` | Sydney | `eu-south-1` | Milan |
| `ap-northeast-1` | Tokyo | `ca-central-1` | Canada |
| `ap-northeast-2` | Seoul | `sa-east-1` | São Paulo |
| `ap-northeast-3` | Osaka | `af-south-1` | Cape Town |
| `ap-south-1` | Mumbai | `me-south-1` | Bahrain |
| `ap-east-1` | Hong Kong | | |

## 🚀 Quick Start

### Prerequisites

1. **AWS China Account** with source S3 bucket
2. **AWS Global Account** with appropriate IAM permissions
3. **AWS CLI** configured for both accounts

### 1. Prepare Target Account

In your target AWS Global account, create an IAM user with S3 permissions:

```bash
# Create IAM user
aws iam create-user --user-name s3-cross-partition-sync

# Create access key
aws iam create-access-key --user-name s3-cross-partition-sync

# Attach S3 policy (replace YOUR-TARGET-BUCKET)
aws iam put-user-policy --user-name s3-cross-partition-sync \
  --policy-name S3SyncPolicy \
  --policy-document file://target-s3-policy.json
```

<details>
<summary>target-s3-policy.json</summary>

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket",
        "s3:GetObjectMetadata",
        "s3:HeadObject",
        "s3:PutObjectAcl",
        "s3:ListMultipartUploadParts",
        "s3:AbortMultipartUpload",
        "s3:ListBucketMultipartUploads"
      ],
      "Resource": [
        "arn:aws:s3:::YOUR-TARGET-BUCKET",
        "arn:aws:s3:::YOUR-TARGET-BUCKET/*"
      ]
    }
  ]
}
```
</details>

### 2. Deploy CloudFormation Stack

```bash
# Clone repository
git clone https://github.com/your-username/s3-cross-partition-sync.git
cd s3-cross-partition-sync

# Deploy stack
aws cloudformation create-stack \
    --stack-name s3-cross-partition-sync \
    --template-body file://s3-cross-account-sync-minimal-template.yaml \
    --parameters \
        ParameterKey=SourceRegion,ParameterValue=cn-northwest-1 \
        ParameterKey=SourceBucketName,ParameterValue=your-source-bucket \
        ParameterKey=TargetRegion,ParameterValue=us-west-2 \
        ParameterKey=TargetBucketName,ParameterValue=your-target-bucket \
        ParameterKey=TargetAccessKeyId,ParameterValue=AKIA... \
        ParameterKey=TargetSecretAccessKey,ParameterValue=... \
    --capabilities CAPABILITY_IAM \
    --region cn-northwest-1
```

### 3. Create Target Bucket

```bash
# In target account, create the bucket
aws s3 mb s3://your-target-bucket --region us-west-2
```

### 4. Test the Sync

```bash
# Upload a file to source bucket
echo "Hello Cross-Partition Sync!" > test-file.txt
aws s3 cp test-file.txt s3://your-source-bucket/ --region cn-northwest-1

# Check if file appears in target bucket
aws s3 ls s3://your-target-bucket/ --region us-west-2
```

## ⚙️ Configuration Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `SourceRegion` | String | `cn-northwest-1` | Source region in AWS China |
| `SourceBucketName` | String | Required | Source S3 bucket name |
| `TargetRegion` | String | `us-west-2` | Target region in AWS Global |
| `TargetBucketName` | String | Required | Target S3 bucket name |
| `TargetAccessKeyId` | String | Required | Target account access key ID |
| `TargetSecretAccessKey` | String | Required | Target account secret access key |
| `SecretsManagerSecretName` | String | `s3-cross-partition-sync-credentials` | Secrets Manager secret name |
| `TmpDirectorySize` | Number | `2048` | Lambda /tmp directory size (MB) |

## 📖 Advanced Usage

### Custom Lambda Configuration

```bash
# Deploy with custom Lambda settings
aws cloudformation create-stack \
    --stack-name s3-sync-custom \
    --template-body file://s3-cross-account-sync-minimal-template.yaml \
    --parameters \
        ParameterKey=SourceRegion,ParameterValue=cn-north-1 \
        ParameterKey=TargetRegion,ParameterValue=eu-west-1 \
        ParameterKey=TmpDirectorySize,ParameterValue=5120 \
        # ... other parameters
    --capabilities CAPABILITY_IAM \
    --region cn-north-1
```

### Manual Testing

```bash
# Get function name from stack outputs
FUNCTION_NAME=$(aws cloudformation describe-stacks \
    --stack-name s3-cross-partition-sync \
    --query 'Stacks[0].Outputs[?OutputKey==`LambdaFunctionName`].OutputValue' \
    --output text \
    --region cn-northwest-1)

# Test with specific file
aws lambda invoke \
    --function-name $FUNCTION_NAME \
    --payload '{"test_key":"path/to/your/file.txt"}' \
    response.json \
    --region cn-northwest-1

# View response
cat response.json
```

### Monitoring and Logs

```bash
# View Lambda logs
aws logs describe-log-streams \
    --log-group-name "/aws/lambda/S3CrossPartitionSync" \
    --region cn-northwest-1

# Get latest log events
aws logs get-log-events \
    --log-group-name "/aws/lambda/S3CrossPartitionSync" \
    --log-stream-name "LATEST_STREAM_NAME" \
    --region cn-northwest-1
```

## 🔧 Troubleshooting

### Common Issues

#### 1. Access Denied Errors

**Problem**: Lambda cannot access target bucket
```
[ERROR] Failed to copy object: An error occurred (403) when calling the HeadObject operation: Forbidden
```

**Solution**: Verify target account IAM permissions and bucket policy

#### 2. Secrets Manager Access Issues

**Problem**: Cannot retrieve credentials
```
[ERROR] Failed to get credentials: An error occurred (AccessDenied) when calling the GetSecretValue operation
```

**Solution**: Check Lambda execution role has `secretsmanager:GetSecretValue` permission

#### 3. Cross-Partition Network Issues

**Problem**: Connection timeouts between China and Global regions
```
[ERROR] Failed to copy object: Read timeout
```

**Solution**: Increase Lambda timeout and retry logic

#### 4. Large File Failures

**Problem**: Out of memory errors for large files
```
[ERROR] Runtime exited with error: signal: killed Runtime.ExitError
```

**Solution**: Increase `TmpDirectorySize` parameter

### Debug Mode

Enable verbose logging by updating the Lambda environment variable:

```bash
aws lambda update-function-configuration \
    --function-name S3CrossPartitionSync \
    --environment Variables='{
        "SOURCE_REGION":"cn-northwest-1",
        "TARGET_REGION":"us-west-2",
        "LOG_LEVEL":"DEBUG"
    }' \
    --region cn-northwest-1
```

## 🔐 Security Considerations

1. **Credentials Storage**: Target account credentials are encrypted at rest in Secrets Manager
2. **Least Privilege**: Lambda execution role has minimal required permissions
3. **Network Security**: Consider VPC endpoints for enhanced security
4. **Credential Rotation**: Regularly rotate target account access keys

## 📊 Performance and Costs

### File Size Handling
- **Small files (< 100MB)**: Processed in Lambda memory
- **Large files (≥ 100MB)**: Processed using Lambda /tmp directory

### Cost Optimization
- **Lambda**: Pay per execution and duration
- **Secrets Manager**: $0.40 per secret per month
- **EventBridge**: First 1M events free per month
- **Data Transfer**: Cross-region transfer charges apply

### Performance Tips
1. **Batch Processing**: Group small files when possible
2. **Concurrent Execution**: Lambda automatically scales
3. **Regional Proximity**: Choose target regions closer to users

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- AWS CloudFormation team for infrastructure as code
- AWS Lambda team for serverless computing
- Community contributors and feedback
---

⭐ **Star this repository if it helped you!**
