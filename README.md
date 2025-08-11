# GitHub Enterprise Server AWS QuickStart

This repository contains AWS CloudFormation templates for deploying GitHub Enterprise Server on Amazon Web Services (AWS) from scratch. The QuickStart provides automated deployment with security best practices and supports two deployment scenarios.

## Architecture Overview

This QuickStart deploys GitHub Enterprise Server with the following components:
- **GitHub Enterprise Server EC2 instance** with current generation instance types (M7i/M7a, C7i/C7a, R7i/R7a)
- **Elastic Block Store (EBS) volumes** with encryption and configurable volume types (GP3, GP2, IO1, IO2)
- **Virtual Private Cloud (VPC)** with public subnet and internet gateway (optional)
- **Security Groups** with GitHub Enterprise required ports
- **Elastic IP** for consistent public IP address
- **IAM roles** for S3 access to license files
- **CloudWatch alarms** for instance monitoring and automatic recovery

## Deployment Options

### Option 1: Deploy into a new VPC (Recommended)
Uses the master template to create all networking infrastructure and GitHub Enterprise Server.

### Option 2: Deploy into an existing VPC
Uses the main template to deploy GitHub Enterprise Server into your existing VPC and subnet.

## Prerequisites

Before deploying, ensure you have:

### 1. AWS Account Setup
- **Active AWS account** with appropriate permissions
- **AWS CLI installed and configured** with your credentials
  ```bash
  # Install AWS CLI (if not already installed)
  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
  unzip awscliv2.zip
  sudo ./aws/install

  # Configure AWS CLI
  aws configure
  ```
- **Required IAM permissions** for the deploying user/role:
  - `ec2:*` (EC2 full access)
  - `cloudformation:*` (CloudFormation full access)
  - `iam:*` (IAM full access)
  - `s3:*` (S3 full access)
  - `secretsmanager:*` (Secrets Manager full access)

### 2. GitHub Enterprise License
- **GitHub Enterprise Server license file** (`.ghl` format)
- Sign up for a trial license at: https://enterprise.github.com/trial
- Download the license file to your local machine
- Valid license is required for GitHub Enterprise Server operation

### 3. AWS Resources Setup
These will be created in the deployment steps below:
- **EC2 Key Pair** for SSH access (with private key stored in Secrets Manager)
- **S3 bucket** to store your GitHub Enterprise license file
- **Secrets Manager secret** to securely store the private key

### 4. Network Planning
- **VPC CIDR block** (if creating new VPC) - recommend `10.0.0.0/16`
- **Access CIDR range** for controlling access to GitHub Enterprise Server
  - For testing: Your public IP + `/32` (most secure)
  - For organization: Your company's IP range
  - **Security Warning**: Avoid `0.0.0.0/0` which allows global access
- Consider your organization's network security requirements

## Supported Regions

GitHub Enterprise Server AMIs are available in the following AWS regions:
- **US**: us-east-1, us-east-2, us-west-1, us-west-2
- **Europe**: eu-west-1, eu-west-2, eu-central-1
- **Asia Pacific**: ap-northeast-1, ap-northeast-2, ap-northeast-3, ap-south-1, ap-southeast-1, ap-southeast-2
- **Canada**: ca-central-1
- **South America**: sa-east-1

## Deployment Steps - Complete Guide

### Step 0: Initial Setup and Verification

1. **Verify AWS CLI Setup**:
   ```bash
   # Check AWS CLI configuration
   aws sts get-caller-identity

   # Set your deployment region (adjust as needed)
   export AWS_REGION=us-east-1
   echo "Using AWS Region: $AWS_REGION"

   # Set the CloudFormation role to assume for deployments
   export CF_ROLE_ARN="arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/CloudFormation-Role"

   # Verify that the role exists
   aws iam get-role --role-name CloudFormation-Role &>/dev/null && \
     echo "✅ CloudFormation role exists: $CF_ROLE_ARN" || \
     echo "❌ CloudFormation role does not exist, please create it first"
   ```

2. **Get Your Public IP for Security**:
   ```bash
   # Get your current public IP for AccessCIDR
   export MY_PUBLIC_IP=$(curl -s https://ipinfo.io/ip)
   echo "Your public IP: $MY_PUBLIC_IP"
   echo "Recommended AccessCIDR: $MY_PUBLIC_IP/32"
   ```

### Step 1: Create and Secure EC2 Key Pair

1. **Create EC2 Key Pair**:
   ```bash
   # Set key pair name
   export KEY_PAIR_NAME="github-enterprise-keypair-$(date +%Y%m%d)"

   # Create key pair and save private key
   aws ec2 create-key-pair \
     --key-name $KEY_PAIR_NAME \
     --query 'KeyMaterial' \
     --output text > ${KEY_PAIR_NAME}.pem

   # Set proper permissions for private key
   chmod 400 ${KEY_PAIR_NAME}.pem

   echo "Key pair created: $KEY_PAIR_NAME"
   echo "Private key saved: ${KEY_PAIR_NAME}.pem"
   ```

2. **Store Private Key in AWS Secrets Manager (Security Best Practice)**:
   ```bash
   # Create secret in AWS Secrets Manager
   aws secretsmanager create-secret \
     --name "github-enterprise/ssh-key/$KEY_PAIR_NAME" \
     --description "Private SSH key for GitHub Enterprise Server" \
     --secret-string file://${KEY_PAIR_NAME}.pem \
     --region $AWS_REGION

   echo "Private key securely stored in AWS Secrets Manager"
   echo "Secret name: github-enterprise/ssh-key/$KEY_PAIR_NAME"
   ```

3. **Secure Local Private Key File**:
   ```bash
   # Optional: Move private key to secure location
   mkdir -p ~/.ssh/github-enterprise
   mv ${KEY_PAIR_NAME}.pem ~/.ssh/github-enterprise/

   # Or remove local copy since it's in Secrets Manager
   # rm ${KEY_PAIR_NAME}.pem  # Uncomment if you want to remove local copy

   echo "Private key secured. You can retrieve it from Secrets Manager if needed:"
   echo "aws secretsmanager get-secret-value --secret-id github-enterprise/ssh-key/$KEY_PAIR_NAME --query SecretString --output text"
   ```

### Step 2: Prepare GitHub Enterprise License

1. **Create S3 Bucket for License Storage**:
   ```bash
   # Set unique bucket name
   export LICENSE_BUCKET="ghe-license-$(aws sts get-caller-identity --query Account --output text)-$(date +%Y%m%d)"

   # Create S3 bucket
   aws s3 mb s3://$LICENSE_BUCKET --region $AWS_REGION

   # Enable versioning (recommended)
   aws s3api put-bucket-versioning \
     --bucket $LICENSE_BUCKET \
     --versioning-configuration Status=Enabled

   # Enable encryption
   aws s3api put-bucket-encryption \
     --bucket $LICENSE_BUCKET \
     --server-side-encryption-configuration '{
       "Rules": [
         {
           "ApplyServerSideEncryptionByDefault": {
             "SSEAlgorithm": "AES256"
           }
         }
       ]
     }'

   echo "S3 bucket created: $LICENSE_BUCKET"
   ```

2. **Upload GitHub Enterprise License**:
   ```bash
   # Ensure you have your license file downloaded
   # Replace 'github-enterprise.ghl' with your actual license filename
   export LICENSE_FILE="github-enterprise.ghl"

   # Verify license file exists
   if [ ! -f "$LICENSE_FILE" ]; then
     echo "ERROR: License file $LICENSE_FILE not found"
     echo "Please download your license from GitHub Enterprise and place it in current directory"
     exit 1
   fi

   # Upload license to S3
   aws s3 cp $LICENSE_FILE s3://$LICENSE_BUCKET/

   echo "License uploaded to S3: s3://$LICENSE_BUCKET/$LICENSE_FILE"
   ```

### Step 3: Prepare CloudFormation Templates

1. **Create S3 Bucket for Templates** (Required for Master Template):
   ```bash
   # The master template requires nested templates to be stored in S3
   # Create a bucket for storing CloudFormation templates
   export TEMPLATES_BUCKET="github-enterprise-templates-$(aws sts get-caller-identity --query Account --output text)-$(date +%Y%m%d)"
   export QS_S3_BUCKET="$TEMPLATES_BUCKET"  # Use your custom bucket or "aws-quickstart"
   export QS_S3_KEY_PREFIX="quickstart-github-enterprise/" # "quickstart-github-enterprise/" for aws-quickstart bucket

   # Create S3 bucket for templates
   aws s3 mb s3://$TEMPLATES_BUCKET --region $AWS_REGION

   # Enable versioning for template management
   aws s3api put-bucket-versioning \
     --bucket $TEMPLATES_BUCKET \
     --versioning-configuration Status=Enabled

   echo "Templates bucket created: $TEMPLATES_BUCKET"
   ```

2. **Upload Templates to S3**:
   ```bash
   # Upload all templates to S3 (required for the master template to work)

   # Upload template files to S3
   aws s3api put-object --bucket $TEMPLATES_BUCKET --key ${QS_S3_KEY_PREFIX}templates/quickstart-github-enterprise-master.template --body templates/quickstart-github-enterprise-master.template
   aws s3api put-object --bucket $TEMPLATES_BUCKET --key ${QS_S3_KEY_PREFIX}templates/quickstart-github-enterprise.template --body templates/quickstart-github-enterprise.template
   aws s3api put-object --bucket $TEMPLATES_BUCKET --key ${QS_S3_KEY_PREFIX}templates/quickstart-github-enterprise-single-az-vpc.template --body templates/quickstart-github-enterprise-single-az-vpc.template

   echo "Templates uploaded and configured for CloudFormation access"
   ```

### Step 4: Set Deployment Parameters

1. **Set Required Parameters**:
   ```bash
   # Required parameters - CUSTOMIZE THESE VALUES
   export STACK_NAME="github-enterprise-stack"
   export SITE_ADMIN_USERNAME="admin"
   export SITE_ADMIN_EMAIL="admin@yourcompany.com"
   export SITE_ADMIN_PASSWORD="YourSecurePassword123"  # Must meet complexity requirements
   export MANAGEMENT_PASSWORD="YourManagementPassword123"  # Must meet complexity requirements
   export ACCESS_CIDR="31.168.164.190/32"  # Office IP only

   # Optional parameters with defaults
   export INSTANCE_TYPE="m7i.xlarge"
   export VOLUME_TYPE="gp3"
   export VOLUME_SIZE="300"
   export PROVISIONED_IOPS="3000"
   export VPC_CIDR="10.0.0.0/16"
   export INITIAL_ORG="initial-organization"
   export INITIAL_REPO="initial-repository"

   # Template-related parameters (set based on your bucket choice above)
   export QS_S3_BUCKET="$TEMPLATES_BUCKET"  # Use your custom bucket or "aws-quickstart"

   echo "Deployment parameters set:"
   echo "  Stack Name: $STACK_NAME"
   echo "  Key Pair: $KEY_PAIR_NAME"
   echo "  License Bucket: $LICENSE_BUCKET"
   echo "  Templates Bucket: $QS_S3_BUCKET"
   echo "  Access CIDR: $ACCESS_CIDR"
   echo "  Instance Type: $INSTANCE_TYPE"
   ```

2. **Validate Parameters**:
   ```bash
   # Check password complexity (minimum requirements)
   if [[ ${#SITE_ADMIN_PASSWORD} -lt 7 ]] || [[ ! "$SITE_ADMIN_PASSWORD" =~ [0-9] ]] || [[ ! "$SITE_ADMIN_PASSWORD" =~ [A-Z] ]]; then
     echo "ERROR: SITE_ADMIN_PASSWORD must be at least 7 characters with at least one number and one uppercase letter"
     exit 1
   fi

   if [[ ${#MANAGEMENT_PASSWORD} -lt 7 ]] || [[ ! "$MANAGEMENT_PASSWORD" =~ [0-9] ]] || [[ ! "$MANAGEMENT_PASSWORD" =~ [A-Z] ]]; then
     echo "ERROR: MANAGEMENT_PASSWORD must be at least 7 characters with at least one number and one uppercase letter"
     exit 1
   fi

   echo "Password requirements validated ✓"
   ```

### Step 5: Deploy GitHub Enterprise Server

Choose your deployment option based on your infrastructure needs:

> **📝 Note about Templates**: The master template uses CloudFormation nested stacks, which require the child templates (quickstart-github-enterprise.template and quickstart-github-enterprise-single-az-vpc.template) to be accessible via S3 URLs. This is why we uploaded templates to S3 in the previous step.

#### Option A: Deploy into New VPC (Recommended for New Setups)

1. **Deploy Full Stack with New VPC**:
   ```bash
   # Deploy using the master template (creates VPC + GitHub Enterprise)
   TEMPLATE_URL="https://${QS_S3_BUCKET}.s3.$AWS_REGION.amazonaws.com/${QS_S3_KEY_PREFIX}templates/quickstart-github-enterprise-master.template"
   aws cloudformation create-stack \
     --stack-name $STACK_NAME \
     --template-url $TEMPLATE_URL \
     --parameters \
       ParameterKey=KeyPairName,ParameterValue=$KEY_PAIR_NAME \
       ParameterKey=AccessCIDR,ParameterValue=$ACCESS_CIDR \
       ParameterKey=VPCCIDR,ParameterValue=$VPC_CIDR \
       ParameterKey=LicenseLocation,ParameterValue=$LICENSE_BUCKET \
       ParameterKey=GHELicense,ParameterValue=$LICENSE_FILE \
       ParameterKey=SiteAdminUsername,ParameterValue="$SITE_ADMIN_USERNAME" \
       ParameterKey=SiteAdminUserEmail,ParameterValue="$SITE_ADMIN_EMAIL" \
       ParameterKey=SiteAdminUserPassword,ParameterValue="$SITE_ADMIN_PASSWORD" \
       ParameterKey=ManagementPassword,ParameterValue="$MANAGEMENT_PASSWORD" \
       ParameterKey=InstanceType,ParameterValue=$INSTANCE_TYPE \
       ParameterKey=VolumeType,ParameterValue=$VOLUME_TYPE \
       ParameterKey=VolumeSize,ParameterValue=$VOLUME_SIZE \
       ParameterKey=ProvisionedIops,ParameterValue=$PROVISIONED_IOPS \
       ParameterKey=InitialOrganization,ParameterValue=$INITIAL_ORG \
       ParameterKey=InitialRepository,ParameterValue=$INITIAL_REPO \
       ParameterKey=QSS3BucketName,ParameterValue=$QS_S3_BUCKET \
       ParameterKey=QSS3BucketRegion,ParameterValue=$AWS_REGION \
       ParameterKey=QSS3KeyPrefix,ParameterValue=$QS_S3_KEY_PREFIX \
     --capabilities CAPABILITY_IAM CAPABILITY_AUTO_EXPAND \
     --role-arn $CF_ROLE_ARN \
     --region $AWS_REGION

   echo "CloudFormation stack deployment initiated: $STACK_NAME"
   echo "Template: quickstart-github-enterprise-master.template"
   ```

2. **Deploy via AWS Console** (Alternative):
   - Navigate to [CloudFormation Console](https://console.aws.amazon.com/cloudformation/)
   - Click "Create Stack" > "With new resources (standard)"
   - Choose "Upload a template file" and select `quickstart-github-enterprise-master.template`
   - Fill in parameters using the values you set above
   - Check "I acknowledge that AWS CloudFormation might create IAM resources"
   - Click "Create stack"

#### Option B: Deploy into Existing VPC

1. **Identify Existing VPC Resources**:
   ```bash
   # List your VPCs
   aws ec2 describe-vpcs --query 'Vpcs[*].[VpcId,CidrBlock,Tags[?Key==`Name`].Value|[0]]' --output table

   # List public subnets in your VPC (replace vpc-xxxxxxxxx with your VPC ID)
   export EXISTING_VPC_ID="vpc-xxxxxxxxx"  # REPLACE WITH YOUR VPC ID
   aws ec2 describe-subnets \
     --filters "Name=vpc-id,Values=$EXISTING_VPC_ID" "Name=map-public-ip-on-launch,Values=true" \
     --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone]' --output table

   # Set the subnet ID (replace with your public subnet ID)
   export EXISTING_SUBNET_ID="subnet-xxxxxxxxx"  # REPLACE WITH YOUR SUBNET ID
   ```

2. **Deploy into Existing VPC**:
   ```bash
   # Deploy using the existing VPC template
   aws cloudformation create-stack \
     --stack-name $STACK_NAME \
     --template-url https://$QS_S3_BUCKET.s3.$AWS_REGION.amazonaws.com/${QS_S3_KEY_PREFIX}templates/quickstart-github-enterprise-master.template \
     --parameters \
       ParameterKey=KeyPairName,ParameterValue=$KEY_PAIR_NAME \
       ParameterKey=VPCID,ParameterValue=$EXISTING_VPC_ID \
       ParameterKey=SubnetId,ParameterValue=$EXISTING_SUBNET_ID \
       ParameterKey=AccessCIDR,ParameterValue=$ACCESS_CIDR \
       ParameterKey=LicenseLocation,ParameterValue=$LICENSE_BUCKET \
       ParameterKey=GHELicense,ParameterValue=$LICENSE_FILE \
       ParameterKey=SiteAdminUsername,ParameterValue="$SITE_ADMIN_USERNAME" \
       ParameterKey=SiteAdminUserEmail,ParameterValue="$SITE_ADMIN_EMAIL" \
       ParameterKey=SiteAdminUserPassword,ParameterValue="$SITE_ADMIN_PASSWORD" \
       ParameterKey=ManagementPassword,ParameterValue="$MANAGEMENT_PASSWORD" \
       ParameterKey=InstanceType,ParameterValue=$INSTANCE_TYPE \
       ParameterKey=VolumeType,ParameterValue=$VOLUME_TYPE \
       ParameterKey=VolumeSize,ParameterValue=$VOLUME_SIZE \
       ParameterKey=InitialOrganization,ParameterValue=$INITIAL_ORG \
       ParameterKey=InitialRepository,ParameterValue=$INITIAL_REPO \
     --capabilities CAPABILITY_IAM \
     --role-arn $CF_ROLE_ARN \
     --region $AWS_REGION

   echo "CloudFormation stack deployment initiated: $STACK_NAME"
   echo "Template: quickstart-github-enterprise.template"
   ```

### Step 6: Monitor Deployment Progress

1. **Monitor Stack Creation**:
   ```bash
   # Check stack status
   aws cloudformation describe-stacks \
     --stack-name $STACK_NAME \
     --role-arn $CF_ROLE_ARN \
     --query 'Stacks[0].StackStatus' \
     --output text

   # Watch stack events in real-time
   aws cloudformation describe-stack-events \
     --stack-name $STACK_NAME \
     --role-arn $CF_ROLE_ARN \
     --query 'StackEvents[0:10].[Timestamp,ResourceStatus,ResourceType,LogicalResourceId]' \
     --output table
   ```

2. **Continuous Monitoring** (Run in separate terminal):
   ```bash
   # Monitor deployment progress (updates every 30 seconds)
   watch -n 30 "aws cloudformation describe-stacks --stack-name $STACK_NAME --role-arn $CF_ROLE_ARN --query 'Stacks[0].StackStatus' --output text"
   ```

3. **View Deployment Progress in Console**:
   - Navigate to [CloudFormation Console](https://console.aws.amazon.com/cloudformation/)
   - Select your stack name: `$STACK_NAME`
   - Click "Events" tab for detailed progress
   - **Expected deployment time: 10-20 minutes**

4. **Check for Deployment Completion**:
   ```bash
   # Wait for stack creation to complete
   aws cloudformation wait stack-create-complete --stack-name $STACK_NAME --role-arn $CF_ROLE_ARN
   echo "Stack creation completed!"

   # Get final stack status
   aws cloudformation describe-stacks \
     --stack-name $STACK_NAME \
     --role-arn $CF_ROLE_ARN \
     --query 'Stacks[0].[StackStatus,CreationTime]' \
     --output table
   ```

### Step 7: Retrieve Deployment Information

1. **Get GitHub Enterprise Server Details**:
   ```bash
   # Get all stack outputs
   aws cloudformation describe-stacks \
     --stack-name $STACK_NAME \
     --role-arn $CF_ROLE_ARN \
     --query 'Stacks[0].Outputs' \
     --output table

   # Get specific values
   export GHE_PUBLIC_IP=$(aws cloudformation describe-stacks \
     --stack-name $STACK_NAME \
     --role-arn $CF_ROLE_ARN \
     --query 'Stacks[0].Outputs[?OutputKey==`PublicIP`].OutputValue' \
     --output text)

   export GHE_URL=$(aws cloudformation describe-stacks \
     --stack-name $STACK_NAME \
     --role-arn $CF_ROLE_ARN \
     --query 'Stacks[0].Outputs[?OutputKey==`GHEURL`].OutputValue' \
     --output text)

   export GHE_INSTANCE_ID=$(aws cloudformation describe-stacks \
     --stack-name $STACK_NAME \
     --role-arn $CF_ROLE_ARN \
     --query 'Stacks[0].Outputs[?OutputKey==`EC2InstanceId`].OutputValue' \
     --output text)

   echo "GitHub Enterprise Server Details:"
   echo "  Public IP: $GHE_PUBLIC_IP"
   echo "  URL: $GHE_URL"
   echo "  Instance ID: $GHE_INSTANCE_ID"
   ```

2. **Retrieve SSH Private Key** (if needed):
   ```bash
   # Retrieve private key from Secrets Manager
   aws secretsmanager get-secret-value \
     --secret-id "github-enterprise/ssh-key/$KEY_PAIR_NAME" \
     --query SecretString \
     --output text > ~/.ssh/github-enterprise/${KEY_PAIR_NAME}.pem

   # Set proper permissions
   chmod 400 ~/.ssh/github-enterprise/${KEY_PAIR_NAME}.pem

   echo "Private key retrieved from Secrets Manager"
   echo "Key location: ~/.ssh/github-enterprise/${KEY_PAIR_NAME}.pem"
   ```

3. **Test SSH Access** (Optional):
   ```bash
   # Test SSH connection to GitHub Enterprise Server
   ssh -i ~/.ssh/github-enterprise/${KEY_PAIR_NAME}.pem \
       -o ConnectTimeout=10 \
       -o StrictHostKeyChecking=no \
       admin@$GHE_PUBLIC_IP \
       "echo 'SSH connection successful'"
   ```

## Parameter Reference

### Required Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `KeyPairName` | EC2 Key Pair for SSH access | `github-enterprise-keypair` |
| `LicenseLocation` | S3 bucket containing license | `your-ghe-license-bucket` |
| `GHELicense` | License filename in S3 bucket | `github-enterprise.ghl` |
| `SiteAdminUsername` | Initial admin username | `admin` |
| `SiteAdminUserEmail` | Admin user email address | `admin@yourcompany.com` |
| `SiteAdminUserPassword` | Admin user password | `SecurePassword123` |
| `ManagementPassword` | Management console password | `ManagementPassword123` |

### Network Parameters

| Parameter | Description | Default | Example |
|-----------|-------------|---------|---------|
| `VPCCIDR` | VPC CIDR block (new VPC only) | `10.0.0.0/16` | `10.0.0.0/16` |
| `AccessCIDR` | IP range for access control | - | `10.0.0.0/8` |
| `VPCID` | Existing VPC ID (existing VPC only) | - | `vpc-12345678` |
| `SubnetId` | Public subnet ID (existing VPC only) | - | `subnet-12345678` |

### Server Configuration Parameters

| Parameter | Description | Default | Options |
|-----------|-------------|---------|---------|
| `InstanceType` | EC2 instance type | `m7i.xlarge` | M7i/M7a, C7i/C7a, R7i/R7a families |
| `VolumeType` | EBS volume type | `gp3` | `gp3`, `gp2`, `io1`, `io2` |
| `VolumeSize` | EBS volume size (GB) | `100` | `100-1000+` |
| `ProvisionedIops` | IOPS for io1/io2 volumes | - | `100-20000` |

### GitHub Configuration Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `InitialOrganization` | Initial organization name | `initial-organization` |
| `InitialRepository` | Initial repository name | `initial-repository` |

## Post-Deployment Configuration

### 1. Access GitHub Enterprise Server

After deployment completes successfully:

1. **Access the Web Interface**:
   ```bash
   # Open GitHub Enterprise Server in your browser
   echo "GitHub Enterprise Server is available at: $GHE_URL"
   echo "Direct IP access: https://$GHE_PUBLIC_IP"

   # For macOS, open automatically
   # open $GHE_URL

   # For Linux with xdg-open
   # xdg-open $GHE_URL
   ```

2. **Initial Web Login**:
   - Navigate to the URL shown above in your browser
   - **Accept the security warning** (self-signed certificate initially)
   - You should see the GitHub Enterprise login page
   - Log in with your admin credentials:
     - **Username**: Value of `$SITE_ADMIN_USERNAME`
     - **Password**: Value of `$SITE_ADMIN_PASSWORD`

3. **SSH Access** (for administration):
   ```bash
   # SSH into the GitHub Enterprise Server
   ssh -i ~/.ssh/github-enterprise/${KEY_PAIR_NAME}.pem admin@$GHE_PUBLIC_IP

   # Or if you kept the key locally:
   # ssh -i ${KEY_PAIR_NAME}.pem admin@$GHE_PUBLIC_IP
   ```

4. **Management Console Access**:
   ```bash
   echo "GitHub Enterprise Management Console: https://$GHE_PUBLIC_IP:8443"
   echo "Management Password: [Use the ManagementPassword you set]"
   ```

### 2. Initial Setup

The QuickStart automatically configures:
- ✅ GitHub Enterprise Server license installation
- ✅ Initial admin user creation
- ✅ Initial organization and repository setup
- ✅ Basic security configuration

### 3. Additional Configuration

Consider these additional setup steps:

1. **Configure custom domain and SSL certificate**
2. **Set up LDAP/SAML authentication** if required
3. **Configure backup strategy** using GitHub Enterprise backup utilities
4. **Set up monitoring and alerting**
5. **Configure GitHub Apps and webhooks** as needed

## Instance Sizing Recommendations

| User Count | Recommended Instance | Volume Type | Volume Size |
|------------|---------------------|-------------|-------------|
| < 100 users | `m7i.large` | `gp3` | 100 GB |
| 100-500 users | `m7i.xlarge` | `gp3` | 200 GB |
| 500-1000 users | `m7i.2xlarge` | `io2` | 500 GB |
| 1000+ users | `m7i.4xlarge+` | `io2` | 1000+ GB |

## Security Considerations

### Network Security
- The security group opens specific ports required for GitHub Enterprise
- Limit `AccessCIDR` to your organization's IP ranges
- Consider using VPN or private connectivity for enhanced security

### Port Configuration
The deployment opens these ports:
- **22** (SSH)
- **80/443** (HTTP/HTTPS)
- **122** (SSH for Git operations)
- **8080/8443** (Management console)
- **9418** (Git protocol)
- **25** (SMTP)
- **1194** (VPN)
- **161** (SNMP)

### Data Security
- EBS volumes are encrypted by default
- Consider enabling additional AWS security services (GuardDuty, Security Hub)
- Implement backup and disaster recovery procedures

## Troubleshooting

### Common Issues

1. **Stack Creation Fails**:
   - Check CloudFormation Events tab for specific error messages
   - Verify all required parameters are provided
   - Ensure IAM permissions are sufficient

2. **License Upload Fails**:
   - Verify S3 bucket permissions
   - Check license file format (.ghl)
   - Ensure bucket and license file names match parameters

3. **Instance Not Accessible**:
   - Check security group rules
   - Verify AccessCIDR parameter
   - Confirm Elastic IP attachment

4. **GitHub Enterprise Setup Fails**:
   - SSH to instance and check `/var/log/cloud-init-output.log`
   - Verify license file validity
   - Check network connectivity to GitHub.com

### Getting Support

- Review CloudFormation stack events for detailed error messages
- Check EC2 instance system logs
- GitHub Enterprise documentation: https://docs.github.com/enterprise-server
- AWS CloudFormation documentation: https://docs.aws.amazon.com/cloudformation/

## Cleanup

To remove all deployed resources safely:

### 1. Backup Important Data (Recommended)
```bash
# Before cleanup, consider backing up:
# - GitHub repositories and data
# - SSH keys and secrets
# - Any custom configurations

# Export important information
echo "Stack Name: $STACK_NAME"
echo "Key Pair Name: $KEY_PAIR_NAME"
echo "License Bucket: $LICENSE_BUCKET"
echo "Templates Bucket: $TEMPLATES_BUCKET"
echo "Secrets Manager Key: github-enterprise/ssh-key/$KEY_PAIR_NAME"
```

### 2. Delete CloudFormation Stack
```bash
# Delete the CloudFormation stack (this removes EC2, VPC, etc.)
aws cloudformation delete-stack --stack-name $STACK_NAME --role-arn $CF_ROLE_ARN

# Wait for deletion to complete
aws cloudformation wait stack-delete-complete --stack-name $STACK_NAME --role-arn $CF_ROLE_ARN
echo "Stack deletion completed"
```

### 3. Clean Up Additional Resources
```bash
# Delete the EC2 Key Pair
aws ec2 delete-key-pair --key-name $KEY_PAIR_NAME
echo "EC2 Key Pair deleted: $KEY_PAIR_NAME"

# Delete the SSH key from Secrets Manager
aws secretsmanager delete-secret \
  --secret-id "github-enterprise/ssh-key/$KEY_PAIR_NAME" \
  --force-delete-without-recovery
echo "SSH key deleted from Secrets Manager"

# Delete S3 buckets and contents (be careful!)
aws s3 rm s3://$LICENSE_BUCKET --recursive
aws s3 rb s3://$LICENSE_BUCKET
echo "S3 license bucket deleted: $LICENSE_BUCKET"

# Delete templates bucket (if you created a custom one)
if [ "$TEMPLATES_BUCKET" != "aws-quickstart" ]; then
  aws s3 rm s3://$TEMPLATES_BUCKET --recursive
  aws s3 rb s3://$TEMPLATES_BUCKET
  echo "S3 templates bucket deleted: $TEMPLATES_BUCKET"
else
  echo "Skipping templates bucket deletion (using aws-quickstart public bucket)"
fi

# Clean up local files
rm -f ~/.ssh/github-enterprise/${KEY_PAIR_NAME}.pem
rm -rf github-enterprise-deployment/
echo "Local cleanup completed"
```

### 4. Verify Cleanup
```bash
# Verify stack is deleted
aws cloudformation describe-stacks --stack-name $STACK_NAME --role-arn $CF_ROLE_ARN 2>&1 | grep -q "does not exist" && echo "✓ Stack deleted" || echo "✗ Stack still exists"

# Verify key pair is deleted
aws ec2 describe-key-pairs --key-names $KEY_PAIR_NAME 2>&1 | grep -q "InvalidKeyPair.NotFound" && echo "✓ Key pair deleted" || echo "✗ Key pair still exists"

# Verify S3 buckets are deleted
aws s3 ls s3://$LICENSE_BUCKET 2>&1 | grep -q "NoSuchBucket" && echo "✓ S3 license bucket deleted" || echo "✗ S3 license bucket still exists"

if [ "$TEMPLATES_BUCKET" != "aws-quickstart" ]; then
  aws s3 ls s3://$TEMPLATES_BUCKET 2>&1 | grep -q "NoSuchBucket" && echo "✓ S3 templates bucket deleted" || echo "✗ S3 templates bucket still exists"
else
  echo "✓ AWS QuickStart bucket (no action needed)"
fi
```

**⚠️ Warning**: This cleanup process will permanently delete your GitHub Enterprise Server and all associated data. Ensure you have proper backups of any important repositories, issues, wikis, and configurations before proceeding.

## Security Best Practices

### During Deployment
- ✅ **Use restrictive AccessCIDR** - Limit to your organization's IP ranges, not `0.0.0.0/0`
- ✅ **Store private keys in Secrets Manager** - Never commit private keys to version control
- ✅ **Use strong passwords** - Follow complexity requirements for admin accounts
- ✅ **Enable S3 encryption** - Encrypt license files and backup data
- ✅ **Use latest instance types** - Deploy with current generation EC2 instances

### Post-Deployment
- 🔒 **Configure SSL certificates** - Replace self-signed certificates with proper SSL
- 🔒 **Set up backup strategy** - Configure GitHub Enterprise backup utilities
- 🔒 **Enable audit logging** - Monitor access and changes
- 🔒 **Configure SAML/LDAP** - Integrate with your identity provider
- 🔒 **Review security settings** - Regularly audit GitHub Enterprise security configuration
- 🔒 **Monitor with CloudWatch** - Set up alerts for system health and security events

### Ongoing Maintenance
- 📊 **Regular backups** - Schedule automated backups
- 🔄 **Security updates** - Keep GitHub Enterprise Server updated
- 📈 **Performance monitoring** - Monitor resource usage and scale as needed
- 🔍 **Security audits** - Regular security reviews and penetration testing

## License

This project is licensed under the Apache License 2.0. See the LICENSE.txt file for details.

## Contributing

This is a GitHub Enterprise Server deployment template. For issues or feature requests, please open an issue in this repository.
