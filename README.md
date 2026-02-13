# Hyperlane Validator AWS CloudFormation

This template provides a "One-Click" production-ready deployment of a Hyperlane Validator on AWS using ECS Fargate.

## Features
- **Serverless**: Uses ECS Fargate for compute (no EC2 instances to manage).
- **Secure**: Uses AWS KMS (Asymmetric ECC_SECG_P256K1) for signing keys.
- **Persistent**: Uses Amazon EFS for the validator database.
- **Durable Storage**: Automatic S3 bucket creation for validator signatures (checkpointing).
- **Networking**: Isolated VPC with public subnets (configurable for private subnets with NAT/VPC Endpoints).

## Architecture
1. **ECS Fargate Task**: Runs the `hyperlane-agent` container.
2. **KMS Key**: Used by the agent to sign merkle roots.
3. **S3 Bucket**: Stores signatures for the Relayer to aggregate.
4. **EFS File System**: Mounts to `/hyperlane_db` for persistent indexing state.
5. **CloudWatch Logs**: Centralized logging for the validator.

## Prerequisites
- AWS Account with appropriate permissions to create IAM roles, ECS services, S3 buckets, and KMS keys.
- A primary RPC URL for the chain you wish to validate.

## Deployment Steps

### 1. Deploy the Template
Click the "Deploy to AWS" button or use the AWS CLI:

```bash
aws cloudformation create-stack \
  --stack-name hyperlane-validator-ethereum \
  --template-body file://hyperlane-validator.yaml \
  --parameters \
    ParameterKey=OriginChainName,ParameterValue=ethereum \
    ParameterKey=RpcUrls,ParameterValue=https://your-rpc-url.com \
  --capabilities CAPABILITY_NAMED_IAM
```

### 2. Fund the Validator
Once deployed, the validator will automatically create a KMS key. You need to fund the address associated with this key so it can announce itself on-chain.

- Find the **KmsKeyArn** in the Stack Outputs.
- The Ethereum address for this KMS key can be derived (the logs will output the address when the validator starts and fails to announce due to lack of funds).
- Send a small amount of gas tokens (e.g., 0.1 ETH/POL/BNB) to that address.

### 3. Verify Operation
- Check the **LogGroup** output to see the validator logs in CloudWatch.
- Verify that JSON files are being created in the **S3BucketName** under the `/<originChainName>` folder.

## Parameters
| Parameter | Description | Default |
|-----------|-------------|---------|
| `OriginChainName` | Name of the chain (e.g., `ethereum`, `polygon`) | - |
| `RpcUrls` | Comma-separated list of RPC URLs | - |
| `ImageTag` | `hyperlane-agent` Docker tag | `agents-v2.0.0` |
| `S3BucketName` | Optional: Use an existing bucket for signatures | (New bucket created) |
| `Cpu` | CPU units (1024 = 1 vCPU) | `1024` |
| `Memory` | Memory in MB | `2048` |

## Maintenance
- **Updates**: Update the `ImageTag` parameter and update the stack to perform a rolling deployment.
- **Monitoring**: Check CloudWatch Metrics for the ECS Service and EFS throughput.
- **Backups**: EFS is automatically backed up if AWS Backup is enabled in your account.
