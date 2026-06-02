# AWS Provisioning

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
# -o overwrites without an interactive "replace?" prompt on a re-run; the
# `|| ... --update` handles "AWS CLI already installed" so a re-run won't error.
unzip -o awscliv2.zip && { sudo ./aws/install || sudo ./aws/install --update; } && aws --version
# (on macOS use the .pkg installer from https://aws.amazon.com/cli/ instead)

# Configure credentials (Access Key ID, Secret Access Key, default region, output format):
aws configure
```

### Step 1: List available regions

Run this and show the full output to the user:

```bash
aws ec2 describe-regions \
  --query 'sort_by(Regions, &RegionName)[].{Region:RegionName}' \
  --output table
```

Ask the user: **"Which AWS region would you like? (e.g. `us-east-1`, `eu-west-1`, `ap-southeast-1`)"**

Re-run `aws configure` if needed to set the chosen region as the default, or export it:

```bash
export AWS_DEFAULT_REGION=<chosen-region>
```

### Step 2: List available instance types

Run this to show common general-purpose types available in the chosen region:

```bash
aws ec2 describe-instance-type-offerings \
  --location-type region \
  --filters "Name=location,Values=$AWS_DEFAULT_REGION" \
  --query 'sort_by(InstanceTypeOfferings, &InstanceType)[?starts_with(InstanceType, `t3`) || starts_with(InstanceType, `t4g`) || starts_with(InstanceType, `m6i`) || starts_with(InstanceType, `c6i`)].InstanceType' \
  --output table
```

For pricing context, common starting points:

| Instance | vCPU | RAM | $/mo (approx) |
|----------|------|-----|----------------|
| `t3.micro` | 2 | 1 GB | ~$8 (free tier eligible) |
| `t3.small` | 2 | 2 GB | ~$15 |
| `t3.medium` | 2 | 4 GB | ~$30 |
| `t3.large` | 2 | 8 GB | ~$60 |
| `m6i.large` | 2 | 8 GB | ~$70 (non-burstable) |

Ask the user: **"Which instance type would you like? (e.g. `t3.small` is a good default for most workloads)"**

### Step 3: Provision

Use `INSTANCE_TYPE` from the user's answer. The AMI is resolved dynamically per-region via SSM so it always picks the latest Ubuntu 24.04 LTS:

```bash
INSTANCE_TYPE=<chosen-instance-type>

# Resolve the latest Ubuntu 24.04 LTS AMI for the current region automatically:
AMI_ID=$(aws ssm get-parameter \
  --name /aws/service/canonical/ubuntu/server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id \
  --query 'Parameter.Value' --output text)
echo "Using AMI: $AMI_ID"

# Create a key pair (skip if you already have one registered):
aws ec2 create-key-pair \
  --key-name vibe-key \
  --query 'KeyMaterial' \
  --output text > ~/.ssh/vibe-key.pem 2>/dev/null || echo "Key pair already exists, continuing"
chmod 600 ~/.ssh/vibe-key.pem 2>/dev/null || true

# Create a security group that allows SSH (22), HTTP (80), HTTPS (443):
SG_ID=$(aws ec2 create-security-group \
  --group-name vibe-sg \
  --description "Vibe server security group" \
  --query 'GroupId' --output text 2>/dev/null || \
  aws ec2 describe-security-groups \
    --filters "Name=group-name,Values=vibe-sg" \
    --query 'SecurityGroups[0].GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id "$SG_ID" --protocol tcp --port 22 --cidr 0.0.0.0/0 2>/dev/null || true
aws ec2 authorize-security-group-ingress --group-id "$SG_ID" --protocol tcp --port 80 --cidr 0.0.0.0/0 2>/dev/null || true
aws ec2 authorize-security-group-ingress --group-id "$SG_ID" --protocol tcp --port 443 --cidr 0.0.0.0/0 2>/dev/null || true

aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type "$INSTANCE_TYPE" \
  --key-name vibe-key \
  --security-group-ids "$SG_ID" \
  --count 1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=vibe-server}]'

SERVER_IP=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=vibe-server" "Name=instance-state-name,Values=running,pending" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)
# IP captured in $SERVER_IP — used by the ssh command below, not echoed in chat.

# AWS Ubuntu AMIs log in as the 'ubuntu' user (Amazon Linux uses 'ec2-user'):
ssh -i ~/.ssh/vibe-key.pem ubuntu@"$SERVER_IP"
```

**If SSH fails**, do not continue to Phase 3B yet. Common causes and fixes:

- *Connection refused / timed out:* the server may still be booting. Wait 30–60s and retry. Confirm the security group allows inbound TCP 22 from your IP.
- *Permission denied (publickey):* you are using the wrong key or the wrong user. Try `ssh -i ~/.ssh/vibe-key.pem ubuntu@"$SERVER_IP"`.
- *No public key registered:* re-check that the key pair name you passed to `--key-name` matches the key on your local machine.
- *Host key changed warning:* if you reused an IP, run `ssh-keygen -R "$SERVER_IP"` then retry.

Resolve SSH before proceeding — all remaining setup happens inside that session.

**Transition to the server:** After provisioning, SSH in using `ssh -i ~/.ssh/vibe-key.pem ubuntu@"$SERVER_IP"`. Once your shell prompt changes (you are now on the server), every Phase 3B command runs there. The `$SERVER_IP` variable lives only in your local shell and is not carried over — that's fine, Phase 3B does not need it.
