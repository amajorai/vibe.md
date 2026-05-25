# AWS Provisioning

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
# -o overwrites without an interactive "replace?" prompt on a re-run; the
# `|| ... --update` handles "AWS CLI already installed" so a re-run won't error.
unzip -o awscliv2.zip && { sudo ./aws/install || sudo ./aws/install --update; } && aws --version
# (on macOS use the .pkg installer from https://aws.amazon.com/cli/ instead)

aws configure
# Enter: Access Key ID, Secret Access Key, region (e.g. us-east-1), output format (json)

# IMPORTANT: AMI IDs are region-specific. The ID below is Ubuntu 22.04 LTS in
# us-east-1 only — it will NOT work in other regions. Look up the current AMI
# for your region first, e.g.:
#   aws ec2 describe-images --owners 099720109477 \
#     --filters 'Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*' \
#               'Name=state,Values=available' \
#     --query 'sort_by(Images,&CreationDate)[-1].ImageId' --output text
# Or use an SSM public parameter (resolves per-region automatically):
#   --image-id resolve:ssm:/aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp2/ami-id

aws ec2 run-instances \
  --image-id ami-0c7217cdde317cfec \
  --instance-type t3.small \
  --key-name <YOUR_KEY_PAIR> \
  --security-group-ids <YOUR_SG_ID> \
  --count 1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=vibe-server}]'

SERVER_IP=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=vibe-server" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)
echo "Server ready at $SERVER_IP"

# AWS Ubuntu AMIs log in as the 'ubuntu' user (Amazon Linux uses 'ec2-user'):
ssh ubuntu@"$SERVER_IP"
```

**If SSH fails**, do not continue to Phase 3B yet. Common causes and fixes:

- *Connection refused / timed out:* the server may still be booting. Wait 30–60s and retry. Confirm the security group allows inbound TCP 22 from your IP.
- *Permission denied (publickey):* you are using the wrong key or the wrong user. Try `ssh -i ~/.ssh/<your-key> ubuntu@"$SERVER_IP"`.
- *No public key registered:* re-check that the key pair name you passed to `--key-name` matches the key on your local machine.
- *Host key changed warning:* if you reused an IP, run `ssh-keygen -R "$SERVER_IP"` then retry.

Resolve SSH before proceeding — all remaining setup happens inside that session.

**Transition to the server:** After provisioning, SSH in using `ssh ubuntu@"$SERVER_IP"`. Once your shell prompt changes (you are now on the server), every Phase 3B command runs there. The `$SERVER_IP` variable lives only in your local shell and is not carried over — that's fine, Phase 3B does not need it.
