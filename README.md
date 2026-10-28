# Secure VPC Web Application

A fully automated AWS infrastructure project built with CloudFormation. It provisions a custom VPC with public and private subnets, deploys a web server on EC2, applies least-privilege IAM access to S3, and sets up CloudWatch monitoring with email alerts, including a billing alarm.

This project was originally built manually through the AWS Console, then rebuilt from scratch as Infrastructure as Code using AWS CloudFormation to demonstrate repeatable, version-controlled infrastructure.

## Architecture

![Architecture diagram](docs/architecture-diagram.png)

The VPC spans two subnet tiers across two availability zones. The public subnet hosts an EC2 instance running Apache, reachable through an internet gateway. A private subnet is reserved for future backend resources and routes outbound traffic through a NAT gateway. The EC2 instance holds an IAM role scoped only to a single S3 bucket, and CloudWatch alarms watch CPU usage, network traffic, and estimated billing, all reporting to one SNS topic that emails alerts.

## Skills demonstrated

- **Networking**: custom VPC, public/private subnet design, route tables, internet gateway, NAT gateway, DNS settings
- **Security**: security groups, network ACLs, least-privilege IAM roles, encrypted S3 storage, restricted SSH access
- **Monitoring**: CloudWatch alarms for CPU, network traffic, and billing, with SNS email notifications
- **Infrastructure as Code**: single-template CloudFormation deployment, parameterized for reuse across environments

## Prerequisites

- An AWS account with billing set up correctly
- AWS CLI installed and configured (`aws configure`)
- An existing EC2 key pair, or create one during setup below
- Your public IP address, for restricting SSH access

## Project structure

```
secure-vpc-web-app/
├── README.md
├── template.yaml
├── docs/
│   └── architecture-diagram.png
└── screenshots/
    ├── stack-resources.png
    └── website-live.png
```

## Parameters reference

| Parameter | Description | Default |
|---|---|---|
| `KeyPairName` | Existing EC2 key pair for SSH access | none, required |
| `InstanceType` | EC2 instance size | `t2.micro` |
| `MyIpAddress` | Your IP in CIDR form, for SSH access | none, required |
| `AlarmEmail` | Email address for CloudWatch alarm notifications | none, required |
| `VpcCidr` | CIDR block for the VPC | `10.0.0.0/16` |

## Full deployment guide

### Step 1: Clone the repository

```
git clone https://github.com/<your-username>/secure-vpc-web-app.git
cd secure-vpc-web-app
```

This gives you `template.yaml`, which describes every AWS resource in the stack, along with this README and the docs/screenshots folders.

### Step 2: Create an EC2 key pair

Skip this step if you already have a key pair you want to reuse. Otherwise:

```
aws ec2 create-key-pair --key-name secure-vpc-key --query 'KeyMaterial' --output text > secure-vpc-key.pem
chmod 400 secure-vpc-key.pem
```

This generates a new SSH key pair in AWS, names it `secure-vpc-key`, and saves the private key locally as `secure-vpc-key.pem`. The `chmod 400` locks the file down so only you can read it, which SSH requires.

Never commit this `.pem` file to Git. Add it to `.gitignore`:

```
*.pem
```

To check if a key pair already exists in your account instead of creating a new one:

```
aws ec2 describe-key-pairs --query 'KeyPairs[*].KeyName' --output table
```

### Step 3: Find your public IP address

```
curl -s https://checkip.amazonaws.com
```

Take the result and add `/32`, for example `203.0.113.25/32`. This value restricts SSH access in the template to your machine only, rather than the entire internet. If your IP changes later (new network, ISP rotation), you'll need to update this parameter and redeploy.

### Step 4: Deploy the stack

```
aws cloudformation create-stack \
  --stack-name secure-vpc-web-app \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=KeyPairName,ParameterValue=secure-vpc-key \
    ParameterKey=MyIpAddress,ParameterValue=102.68.79.14/32 \
    ParameterKey=AlarmEmail,ParameterValue=you@example.com
```

Replace the parameter values with your own key pair name, IP address, and email. The `--capabilities CAPABILITY_NAMED_IAM` flag is required because the template creates an IAM role.

This kicks off deployment in the background. The NAT gateway is the slowest resource to provision, so full stack creation typically takes 5 to 10 minutes.

### Step 5: Track deployment progress

```
aws cloudformation describe-stacks --stack-name secure-vpc-web-app --query 'Stacks[0].StackStatus'
```

Keep running this until it returns `CREATE_COMPLETE`. If it instead returns `ROLLBACK_COMPLETE`, something failed. See exactly what with:

```
aws cloudformation describe-stack-events --stack-name secure-vpc-web-app --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'
```

### Step 6: Retrieve your stack outputs

```
aws cloudformation describe-stacks --stack-name secure-vpc-web-app --query 'Stacks[0].Outputs'
```

This returns the web server's public URL, the S3 bucket name, the VPC ID, and the SNS topic ARN. Visit the URL in a browser to confirm the site is live.

### Step 7: Confirm your SNS email subscription

Check the inbox for the address you set as `AlarmEmail`. AWS sends a confirmation link, and alarms will not notify you until you click it.

### Step 8: Enable billing alerts (manual, one-time)

CloudFormation cannot toggle this account-level setting. In the AWS Console:

1. Click your account name (top right) → **Billing and Cost Management**
2. Left sidebar → **Billing Preferences**
3. Check **"Receive CloudWatch Billing Alerts"**
4. Click **Save preferences**

Without this, the billing metric never publishes and the billing alarm sits with no data.

### Step 9: Take screenshots for documentation

While the stack is running, capture:
- The CloudFormation stack's resource list (Console → CloudFormation → your stack → Resources tab)
- The live website in your browser
- The VPC resource map (Console → VPC → Resource Map)

Save these into the `screenshots/` folder.

### Step 10: Push to GitHub

If you haven't created the GitHub repo yet, do that first at github.com (New repository, leave it empty, don't initialize with a README).

```
git init
git add .
git commit -m "Add Secure VPC Web Application CloudFormation template"
git branch -M main
git remote add origin https://github.com/<your-username>/secure-vpc-web-app.git
git push -u origin main
```

### Step 11: Update the stack later (e.g. adding a new alarm)

Change `template.yaml`, then run `update-stack` instead of `create-stack`. To keep existing parameter values unchanged:

```
aws cloudformation update-stack \
  --stack-name secure-vpc-web-app \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=KeyPairName,UsePreviousValue=true \
    ParameterKey=MyIpAddress,UsePreviousValue=true \
    ParameterKey=AlarmEmail,UsePreviousValue=true
```

CloudFormation compares the updated template against the live stack and only touches what changed.

## Monitoring and alerts

Three CloudWatch alarms are configured, all reporting to a single SNS topic:

- **High CPU alarm**: triggers above 70% average CPU over 10 minutes
- **Network traffic spike alarm**: triggers on unusually high inbound traffic
- **Billing alarm**: triggers when estimated charges exceed $0.90 for the current billing period

## Cost considerations

This stack is not fully free. The main cost driver is the **NAT gateway**, which charges hourly plus data processing fees regardless of EC2 usage, since it has no free tier allowance. The `t2.micro` EC2 instance itself is Free Tier eligible for 750 hours a month on accounts within their first 12 months.

To pause the EC2 instance without deleting the stack:

```
aws ec2 stop-instances --instance-ids <your-instance-id>
aws ec2 start-instances --instance-ids <your-instance-id>
```

Stopping and restarting assigns a new public IP unless an Elastic IP is attached. Note the NAT gateway keeps charging even while the instance is stopped, since it has no "paused" state.

To fully tear down and stop all charges:

```
aws cloudformation delete-stack --stack-name secure-vpc-web-app
```

## Troubleshooting notes

Real issues encountered while building this, kept here for reference:

- **Apache not installed after boot**: the original template used `dnf`, which only works on Amazon Linux 2023. If the AMI resolves to Amazon Linux 2 instead, `dnf` fails silently and Apache never installs. This template uses `yum`, which works on both versions.
- **Site unreachable after a stop/start cycle**: EC2 releases the public IP on stop and assigns a new one on start, unless an Elastic IP is attached. Always re-check the current public IP after restarting.
- **"Connection refused" vs "connection timed out"**: a timeout usually points to a security group or NACL issue. "Connection refused" means the network path is fine but nothing is listening on the port, usually meaning the web server process itself isn't running. Check with `sudo systemctl status httpd` over SSH.
- **VS Code flags `!Ref`, `!Sub`, etc. as errors**: this happens when a CloudFormation-aware extension (like AWS Toolkit) is disabled, leaving a generic YAML parser that doesn't understand short-form intrinsic functions. Enable AWS Toolkit, or add CloudFormation custom tags to VS Code's `yaml.customTags` setting.

## Security highlights

- SSH access restricted to a single IP address, never open to the internet
- IAM role scoped to read/write/list on exactly one S3 bucket, nothing broader
- S3 bucket encrypted by default with public access fully blocked
- Network ACLs enforced at the subnet level in addition to security groups
- No hardcoded credentials anywhere in the template, all access is handled through IAM roles

## Cleanup

Always delete the stack when finished testing, to avoid ongoing charges:

```
aws cloudformation delete-stack --stack-name secure-vpc-web-app
```

## Author

Julia
