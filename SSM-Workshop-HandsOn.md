# AWS Systems Manager Session Manager - Hands-On Workshop
## Secure Access to Private EC2 Instances Without SSH Keys

---

## Workshop Overview

Learn how to securely access EC2 instances in private subnets using AWS Systems Manager Session Manager **without SSH keys, NAT Gateway, or bastion hosts**. You'll manually configure all components including security groups, VPC endpoints, IAM roles, and access both Linux and Windows instances (including Windows GUI via RDP).

**What You'll Build:**
- ✅ Linux instance access via Session Manager
- ✅ Windows instance access via Session Manager
- ✅ Windows GUI (RDP) access via port forwarding
- ✅ VPC Endpoints for private connectivity
- ✅ IAM roles for SSM access
- ✅ Security groups for proper isolation

**Duration:** 90-120 minutes  
**Cost:** ~$2-3 for workshop duration  
**Level:** Intermediate

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    AWS Cloud                             │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │         VPC (10.0.0.0/16) - Part 1 (CFN)          │ │
│  │                                                    │ │
│  │  ┌──────────────────────────────────────────────┐ │ │
│  │  │   Private Subnet (10.0.1.0/24)               │ │ │
│  │  │                                              │ │ │
│  │  │  ┌─────────────┐      ┌─────────────┐      │ │ │
│  │  │  │   Linux     │      │  Windows    │      │ │ │
│  │  │  │  Instance   │      │  Instance   │      │ │ │
│  │  │  │ (Part 2)    │      │ (Part 2)    │      │ │ │
│  │  │  └──────┬──────┘      └──────┬──────┘      │ │ │
│  │  │         │                    │             │ │ │
│  │  └─────────┼────────────────────┼─────────────┘ │ │
│  │            │                    │               │ │
│  │            │  ┌─────────────────┴──────────┐   │ │
│  │            └──┤   VPC Endpoints (Part 2)   │   │ │
│  │               │  • ssm                     │   │ │
│  │               │  • ssmmessages             │   │ │
│  │               │  • ec2messages             │   │ │
│  │               │  • s3 (gateway)            │   │ │
│  │               └────────────────────────────┘   │ │
│  └────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

---

## Prerequisites

- AWS Account with admin access
- AWS CLI installed and configured
- Session Manager plugin installed
- Basic understanding of VPC, EC2, and IAM

### Install Session Manager Plugin

**macOS:**
```bash
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/mac/sessionmanager-bundle.zip" -o "sessionmanager-bundle.zip"
unzip sessionmanager-bundle.zip
sudo ./sessionmanager-bundle/install -i /usr/local/sessionmanagerplugin -b /usr/local/bin/session-manager-plugin
```

**Linux:**
```bash
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/linux_64bit/session-manager-plugin.rpm" -o "session-manager-plugin.rpm"
sudo yum install -y session-manager-plugin.rpm
```

**Windows:**
Download and install from: https://s3.amazonaws.com/session-manager-downloads/plugin/latest/windows/SessionManagerPluginSetup.exe

---

## Part 1: Deploy VPC Infrastructure (10 minutes)

### Step 1: Deploy CloudFormation Template

**Via AWS Console:**
1. Navigate to **CloudFormation Console**
2. Click **Create stack** → **With new resources**
3. Upload `ssm-vpc-infrastructure.yaml`
4. Stack name: `ssm-workshop-vpc`
5. Click **Next** → **Next** → **Create stack**
6. Wait for **CREATE_COMPLETE** status (~2 minutes)

**Via AWS CLI:**
```bash
aws cloudformation create-stack \
  --stack-name ssm-workshop-vpc \
  --template-body file://ssm-vpc-infrastructure.yaml \
  --region us-east-1

# Wait for completion
aws cloudformation wait stack-create-complete \
  --stack-name ssm-workshop-vpc
```

### Step 2: Note Stack Outputs

```bash
aws cloudformation describe-stacks \
  --stack-name ssm-workshop-vpc \
  --query 'Stacks[0].Outputs' \
  --output table
```

**Save these values:**
- VPCId: `vpc-xxxxx`
- PrivateSubnetId: `subnet-xxxxx`
- PrivateRouteTableId: `rtb-xxxxx`

---

## Part 2: Manual Configuration (Hands-On)

---

## Module 1: Create IAM Role for EC2 Instances (15 minutes)

### Step 1: Create IAM Role

1. Navigate to **IAM Console** → **Roles**
2. Click **Create role**
3. Select **AWS service** → **EC2**
4. Click **Next**

### Step 2: Attach Policy

1. Search for: `AmazonSSMManagedInstanceCore`
2. Check the box next to it
3. Click **Next**

### Step 3: Name and Create

1. Role name: `EC2-SSM-Role`
2. Description: `Allows EC2 instances to use Systems Manager`
3. Click **Create role**

**Via CLI (Alternative):**
```bash
# Create trust policy
cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create role
aws iam create-role \
  --role-name EC2-SSM-Role \
  --assume-role-policy-document file://trust-policy.json

# Attach policy
aws iam attach-role-policy \
  --role-name EC2-SSM-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Create instance profile
aws iam create-instance-profile \
  --instance-profile-name EC2-SSM-InstanceProfile

# Add role to instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-SSM-InstanceProfile \
  --role-name EC2-SSM-Role
```

---

## Module 2: Create Security Groups (15 minutes)

### Step 1: Create Security Group for VPC Endpoints

1. Navigate to **EC2 Console** → **Security Groups**
2. Click **Create security group**

**Settings:**
- Name: `VPCEndpoint-SG`
- Description: `Security group for VPC endpoints`
- VPC: Select `SSM-Workshop-VPC`

**Inbound Rules:**
- Type: `HTTPS`
- Protocol: `TCP`
- Port: `443`
- Source: `10.0.0.0/16` (VPC CIDR)
- Description: `Allow HTTPS from VPC`

3. Click **Create security group**
4. **Note the Security Group ID:** `sg-xxxxx`

### Step 2: Create Security Group for EC2 Instances

1. Click **Create security group**

**Settings:**
- Name: `Instance-SG`
- Description: `Security group for EC2 instances`
- VPC: Select `SSM-Workshop-VPC`

**Outbound Rules:**
- Type: `HTTPS`
- Protocol: `TCP`
- Port: `443`
- Destination: Select `VPCEndpoint-SG` (the security group you just created)
- Description: `Allow HTTPS to VPC endpoints`

2. Click **Create security group**
3. **Note the Security Group ID:** `sg-yyyyy`

**Via CLI (Alternative):**
```bash
VPC_ID=$(aws cloudformation describe-stacks \
  --stack-name ssm-workshop-vpc \
  --query 'Stacks[0].Outputs[?OutputKey==`VPCId`].OutputValue' \
  --output text)

# Create VPC Endpoint security group
ENDPOINT_SG=$(aws ec2 create-security-group \
  --group-name VPCEndpoint-SG \
  --description "Security group for VPC endpoints" \
  --vpc-id ${VPC_ID} \
  --output text)

# Add inbound rule
aws ec2 authorize-security-group-ingress \
  --group-id ${ENDPOINT_SG} \
  --protocol tcp \
  --port 443 \
  --cidr 10.0.0.0/16

# Create Instance security group
INSTANCE_SG=$(aws ec2 create-security-group \
  --group-name Instance-SG \
  --description "Security group for EC2 instances" \
  --vpc-id ${VPC_ID} \
  --output text)

# Add outbound rule to VPC endpoints
aws ec2 authorize-security-group-egress \
  --group-id ${INSTANCE_SG} \
  --protocol tcp \
  --port 443 \
  --source-group ${ENDPOINT_SG}
```

---

## Module 3: Create VPC Endpoints (20 minutes)

### Step 1: Create SSM VPC Endpoint

1. Navigate to **VPC Console** → **Endpoints**
2. Click **Create endpoint**

**Settings:**
- Name: `ssm-endpoint`
- Service category: `AWS services`
- Service name: Search for `com.amazonaws.us-east-1.ssm`
- VPC: Select `SSM-Workshop-VPC`
- Subnets: Select `SSM-Private-Subnet`
- Security groups: Select `VPCEndpoint-SG`
- Enable DNS name: ✅ **Checked**

3. Click **Create endpoint**

### Step 2: Create SSM Messages VPC Endpoint

1. Click **Create endpoint**

**Settings:**
- Name: `ssmmessages-endpoint`
- Service name: Search for `com.amazonaws.us-east-1.ssmmessages`
- VPC: Select `SSM-Workshop-VPC`
- Subnets: Select `SSM-Private-Subnet`
- Security groups: Select `VPCEndpoint-SG`
- Enable DNS name: ✅ **Checked**

2. Click **Create endpoint**

### Step 3: Create EC2 Messages VPC Endpoint

1. Click **Create endpoint**

**Settings:**
- Name: `ec2messages-endpoint`
- Service name: Search for `com.amazonaws.us-east-1.ec2messages`
- VPC: Select `SSM-Workshop-VPC`
- Subnets: Select `SSM-Private-Subnet`
- Security groups: Select `VPCEndpoint-SG`
- Enable DNS name: ✅ **Checked**

2. Click **Create endpoint**

### Step 4: Create S3 Gateway Endpoint

1. Click **Create endpoint**

**Settings:**
- Name: `s3-endpoint`
- Service category: `AWS services`
- Service name: Search for `com.amazonaws.us-east-1.s3` (Type: Gateway)
- VPC: Select `SSM-Workshop-VPC`
- Route tables: Select `SSM-Private-RouteTable`

2. Click **Create endpoint**

**Wait for all endpoints to show "Available" status (~5 minutes)**

**Via CLI (Alternative):**
```bash
SUBNET_ID=$(aws cloudformation describe-stacks \
  --stack-name ssm-workshop-vpc \
  --query 'Stacks[0].Outputs[?OutputKey==`PrivateSubnetId`].OutputValue' \
  --output text)

RTB_ID=$(aws cloudformation describe-stacks \
  --stack-name ssm-workshop-vpc \
  --query 'Stacks[0].Outputs[?OutputKey==`PrivateRouteTableId`].OutputValue' \
  --output text)

# Create SSM endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id ${VPC_ID} \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.ssm \
  --subnet-ids ${SUBNET_ID} \
  --security-group-ids ${ENDPOINT_SG}

# Create SSM Messages endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id ${VPC_ID} \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.ssmmessages \
  --subnet-ids ${SUBNET_ID} \
  --security-group-ids ${ENDPOINT_SG}

# Create EC2 Messages endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id ${VPC_ID} \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.ec2messages \
  --subnet-ids ${SUBNET_ID} \
  --security-group-ids ${ENDPOINT_SG}

# Create S3 Gateway endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id ${VPC_ID} \
  --vpc-endpoint-type Gateway \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids ${RTB_ID}
```

---

## Module 4: Launch Linux Instance (15 minutes)

### Step 1: Launch Instance

1. Navigate to **EC2 Console** → **Instances**
2. Click **Launch instances**

**Settings:**
- Name: `SSM-Linux-Instance`
- AMI: **Amazon Linux 2023** (or Amazon Linux 2)
- Instance type: `t3.micro`
- Key pair: **Proceed without a key pair** (Don't create one!)

**Network settings:**
- VPC: Select `SSM-Workshop-VPC`
- Subnet: Select `SSM-Private-Subnet`
- Auto-assign public IP: **Disable**
- Security group: Select existing → `Instance-SG`

**Advanced details:**
- IAM instance profile: Select `EC2-SSM-InstanceProfile`

**User data** (paste in User data field):
```bash
#!/bin/bash
yum update -y
yum install -y amazon-ssm-agent
systemctl enable amazon-ssm-agent
systemctl start amazon-ssm-agent
```

3. Click **Launch instance**

### Step 2: Verify Instance Registration

Wait 3-5 minutes, then:

1. Navigate to **Systems Manager Console** → **Fleet Manager**
2. You should see `SSM-Linux-Instance` listed
3. Status should show **Online**

**Via CLI:**
```bash
# Get instance ID
LINUX_INSTANCE=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=SSM-Linux-Instance" "Name=instance-state-name,Values=running" \
  --query 'Reservations[0].Instances[0].InstanceId' \
  --output text)

# Check SSM registration
aws ssm describe-instance-information \
  --filters "Key=InstanceIds,Values=${LINUX_INSTANCE}"
```

---

## Module 5: Launch Windows Instance (15 minutes)

### Step 1: Launch Instance

1. Navigate to **EC2 Console** → **Instances**
2. Click **Launch instances**

**Settings:**
- Name: `SSM-Windows-Instance`
- AMI: **Windows Server 2022 Base**
- Instance type: `t3.medium` (Windows needs more resources)
- Key pair: **Proceed without a key pair**

**Network settings:**
- VPC: Select `SSM-Workshop-VPC`
- Subnet: Select `SSM-Private-Subnet`
- Auto-assign public IP: **Disable**
- Security group: Select existing → `Instance-SG`

**Advanced details:**
- IAM instance profile: Select `EC2-SSM-InstanceProfile`

**User data** (paste in User data field):
```powershell
<powershell>
# Ensure SSM Agent is running
$ssmService = Get-Service -Name AmazonSSMAgent -ErrorAction SilentlyContinue
if ($ssmService) {
    Start-Service AmazonSSMAgent
    Set-Service AmazonSSMAgent -StartupType Automatic
}

# Enable RDP for Session Manager Port Forwarding
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"

Write-Host "SSM Agent configured successfully"
</powershell>
```

2. Click **Launch instance**

### Step 2: Verify Instance Registration

Wait 5-7 minutes (Windows takes longer), then:

1. Navigate to **Systems Manager Console** → **Fleet Manager**
2. You should see `SSM-Windows-Instance` listed
3. Status should show **Online**

---

## Module 6: Access Linux Instance via Session Manager (10 minutes)

### Step 1: Start Session via Console

1. Navigate to **Systems Manager Console**
2. Go to **Session Manager** → **Start session**
3. Select `SSM-Linux-Instance`
4. Click **Start session**

You'll get a terminal in your browser!

### Step 2: Run Commands

```bash
# Check user
whoami
# Output: ssm-user

# Check instance metadata
curl http://169.254.169.254/latest/meta-data/instance-id

# Check network
ip addr show

# Verify no internet access (should fail)
ping -c 3 google.com

# Check SSM agent
sudo systemctl status amazon-ssm-agent

# Become root
sudo su -

# Exit
exit
```

### Step 3: Start Session via CLI

```bash
# Get instance ID
LINUX_INSTANCE=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=SSM-Linux-Instance" "Name=instance-state-name,Values=running" \
  --query 'Reservations[0].Instances[0].InstanceId' \
  --output text)

# Start session
aws ssm start-session --target ${LINUX_INSTANCE}
```

---

## Module 7: Access Windows Instance via Session Manager (10 minutes)

### Step 1: Start PowerShell Session

1. Navigate to **Systems Manager Console**
2. Go to **Session Manager** → **Start session**
3. Select `SSM-Windows-Instance`
4. Click **Start session**

You'll get a PowerShell terminal!

### Step 2: Run Commands

```powershell
# Check user
whoami
# Output: nt authority\system

# Check computer info
Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion

# Check instance metadata
Invoke-RestMethod -Uri http://169.254.169.254/latest/meta-data/instance-id

# Check network
Get-NetIPAddress

# Verify no internet (should fail)
Test-NetConnection google.com -Port 443

# Check SSM agent
Get-Service AmazonSSMAgent

# Check RDP status
Get-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections"

# Exit
exit
```

---

## Module 8: Windows GUI Access via RDP Port Forwarding (20 minutes)

### Step 1: Set Administrator Password

**Via Session Manager Console:**
1. Start session to Windows instance
2. Run this command:
```powershell
net user Administrator MySecurePass123!
```

**Via CLI:**
```bash
WINDOWS_INSTANCE=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=SSM-Windows-Instance" "Name=instance-state-name,Values=running" \
  --query 'Reservations[0].Instances[0].InstanceId' \
  --output text)

aws ssm send-command \
  --instance-ids ${WINDOWS_INSTANCE} \
  --document-name "AWS-RunPowerShellScript" \
  --parameters 'commands=["net user Administrator MySecurePass123!"]'
```

### Step 2: Start Port Forwarding

**Open a new terminal and run:**
```bash
aws ssm start-session \
  --target ${WINDOWS_INSTANCE} \
  --document-name AWS-StartPortForwardingSession \
  --parameters "portNumber=3389,localPortNumber=13389"
```

**Expected output:**
```
Starting session with SessionId: your-session-id
Port 13389 opened for sessionId your-session-id.
Waiting for connections...
```

**Keep this terminal open!**

### Step 3: Connect via RDP

**Windows:**
1. Open **Remote Desktop Connection** (mstsc.exe)
2. Computer: `localhost:13389`
3. Username: `Administrator`
4. Password: `MySecurePass123!`
5. Click **Connect**

**macOS:**
1. Open **Microsoft Remote Desktop**
2. Add PC: `localhost:13389`
3. Username: `Administrator`
4. Password: `MySecurePass123!`
5. Connect

**Linux:**
```bash
rdesktop localhost:13389 -u Administrator -p MySecurePass123!
# or
xfreerdp /v:localhost:13389 /u:Administrator /p:MySecurePass123!
```

### Step 4: Verify GUI Access

Once connected:
1. ✅ Open **Server Manager**
2. ✅ Open **PowerShell**
3. ✅ Run: `Get-ComputerInfo`
4. ✅ Try to open Internet Explorer (should fail - no internet)
5. ✅ Verify you're in a private subnet

---

## Validation and Testing

### Test 1: Verify No Internet Access

**Linux:**
```bash
aws ssm start-session --target ${LINUX_INSTANCE}
ping -c 3 8.8.8.8  # Should fail
curl https://google.com  # Should fail
```

**Windows:**
```powershell
aws ssm start-session --target ${WINDOWS_INSTANCE}
Test-NetConnection google.com -Port 443  # Should fail
```

### Test 2: Verify VPC Endpoints

```bash
aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=${VPC_ID}" \
  --query 'VpcEndpoints[*].[ServiceName,State]' \
  --output table
```

All should show **available** status.

### Test 3: Session Manager Audit

```bash
# List active sessions
aws ssm describe-sessions --state Active

# List session history
aws ssm describe-sessions --state History --max-results 10
```

---

## Troubleshooting

### Issue 1: Instance Not Showing in Session Manager

**Check:**
1. IAM role attached to instance?
2. VPC endpoints in "available" state?
3. Security groups allow HTTPS (443)?
4. Wait 5-10 minutes after launch

**Solution:**
```bash
# Check instance profile
aws ec2 describe-instances --instance-ids ${LINUX_INSTANCE} \
  --query 'Reservations[0].Instances[0].IamInstanceProfile'

# Check VPC endpoints
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=${VPC_ID}"

# Restart SSM agent (via Run Command)
aws ssm send-command \
  --instance-ids ${LINUX_INSTANCE} \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["sudo systemctl restart amazon-ssm-agent"]'
```

### Issue 2: Port Forwarding Fails

**Check:**
1. Session Manager plugin installed?
2. Port 13389 not already in use?

**Solution:**
```bash
# Verify plugin
session-manager-plugin --version

# Check port
netstat -an | grep 13389

# Try different port
aws ssm start-session \
  --target ${WINDOWS_INSTANCE} \
  --document-name AWS-StartPortForwardingSession \
  --parameters "portNumber=3389,localPortNumber=23389"
```

### Issue 3: Windows Password Not Working

**Solution:**
```bash
# Reset password
aws ssm send-command \
  --instance-ids ${WINDOWS_INSTANCE} \
  --document-name "AWS-RunPowerShellScript" \
  --parameters 'commands=["net user Administrator NewPassword123!"]'
```

---

## Cleanup

### Step 1: Terminate Instances

1. Navigate to **EC2 Console** → **Instances**
2. Select both instances
3. **Instance state** → **Terminate instance**

### Step 2: Delete VPC Endpoints

1. Navigate to **VPC Console** → **Endpoints**
2. Select all 4 endpoints
3. **Actions** → **Delete VPC endpoints**

### Step 3: Delete Security Groups

1. Navigate to **EC2 Console** → **Security Groups**
2. Delete `Instance-SG`
3. Delete `VPCEndpoint-SG`

### Step 4: Delete IAM Resources

1. Navigate to **IAM Console** → **Roles**
2. Delete `EC2-SSM-Role`
3. Go to **Instance Profiles**
4. Delete `EC2-SSM-InstanceProfile`

### Step 5: Delete CloudFormation Stack

```bash
aws cloudformation delete-stack --stack-name ssm-workshop-vpc
```

---

## Key Takeaways

✅ **No SSH keys needed** - Session Manager provides keyless access  
✅ **No bastion hosts** - Direct access via VPC endpoints  
✅ **No NAT Gateway** - Saves ~$32/month  
✅ **Audit trail** - All sessions logged in CloudTrail  
✅ **IAM-based access** - Leverage existing IAM policies  
✅ **Port forwarding** - Access RDP, databases, any TCP port  
✅ **Private connectivity** - Traffic stays within AWS network  

---

## Additional Resources

- [AWS Systems Manager Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
- [VPC Endpoints for Systems Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/setup-create-vpc.html)
- [Port Forwarding with Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-sessions-start.html#sessions-start-port-forwarding)

---

**Workshop Version:** 1.0  
**Last Updated:** January 2026  
**License:** MIT
