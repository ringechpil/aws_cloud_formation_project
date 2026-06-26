# AWS CloudFormation — WordPress on EC2

A hands-on project that provisions a WordPress site on AWS using CloudFormation (Infrastructure as Code). Each step builds on the previous one, progressively automating more of the setup.

---

## What Gets Deployed

- **VPC** with 2 public and 2 private subnets across 2 availability zones
- **Internet Gateway** and route tables
- **EC2 instance** (t3.micro, Amazon Linux 2023) in the public subnet
- **LAMP stack** (Apache, PHP, MariaDB, WordPress) auto-installed via UserData
- **Security groups** for the web server, load balancer, ASG, and database
- **Application Load Balancer** (internet-facing) across both public subnets, along with:
  - *Target Group* with health checks against `/health.html`
  - *HTTP Listener* on port 80
- **Launch Template** — reusable EC2 blueprint with full WordPress UserData bootstrap
- **Auto Scaling Group** — min 2, desired 2, max 4 instances across both public subnets

The infrastructure is split across three templates:

| Template | Responsibility |
|----------|---------------|
| `templates/network-security.yaml` | VPC, subnets, Internet Gateway, route tables, all security groups |
| `templates/wordpress-server.yaml` | EC2 instance with automated LAMP stack and WordPress installation |
| `templates/autoscaling-alb.yaml` | ALB, Target Group, Listener, Launch Template, and Auto Scaling Group |

Each template imports values exported by `network-security.yaml`, so the network stack must always be deployed first.

---

## Prerequisites

- AWS CLI configured with appropriate credentials
- A key pair named `vockey` (or update `KeyPairName` in the template)
- A `.pem` file for SSH access (`labsuser.pem`)
- Your local machine's public IP address

---

## Step-by-Step Guide

### 1. Deploy the Network and Security Groups

This creates the VPC, subnets, Internet Gateway, route tables, and all security groups. Pass your local IP so SSH access is locked to your machine.

```bash
aws cloudformation deploy \
  --stack-name network-stack \
  --template-file templates/network-security.yaml \
  --parameter-overrides MyIpAddress=<your-ip>
```

To inspect deployed resources at any point:

```bash
aws cloudformation describe-stack-resources \
  --stack-name <stack-name> \
  --query 'StackResources[*].[ResourceType,LogicalResourceId,ResourceStatus]' \
  --output table
```

---

### 2. Deploy the Web Server

Once the network stack is up, deploy the EC2 instance. It will automatically pull the subnet and security group IDs from the network stack's exports.

```bash
aws cloudformation deploy \
  --stack-name wordpress-stack \
  --template-file templates/wordpress-server.yaml
```

Check instance details:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:aws:cloudformation:stack-name,Values=wordpress-stack" \
  --query "Reservations[*].Instances[*].{Name:Tags[?Key=='Name']|[0].Value,Status:State.Name,PublicIp:PublicIpAddress,PrivateIp:PrivateIpAddress,Subnet:SubnetId,SecurityGroup:SecurityGroups[0].GroupName,AMI:ImageId,InstanceType:InstanceType}" \
  --output table
```

---

### 3. Connect to the Instance

Connect via SSH:

```bash
ssh -i labsuser.pem ec2-user@<instance-public-ip>
```

---

### 4. Install WordPress Manually

See the full step-by-step guide in `docs/deployment-guide.md`.

---

### 5. Access WordPress

Open a browser and navigate to:

```
http://<instance-public-ip>
```

Complete the WordPress setup wizard to create your site.

---

### 6. Automate the Installation (UserData)

The manual steps above are already automated in the `UserData` section of `templates/wordpress-server.yaml`. When the stack is deployed, the EC2 instance bootstraps itself — no SSH required.

To verify WordPress is running after deployment, visit:

```
http://<instance-public-ip>/health.html
```

You should see: **WordPress Installation Completed Successfully**

---

### 7. Deploy the Application Load Balancer

With the network stack in place, deploy the ALB stack. It creates an internet-facing Application Load Balancer across both public subnets, a Target Group, and an HTTP Listener.

```bash
aws cloudformation deploy \
  --stack-name alb-stack \
  --template-file templates/autoscaling-alb.yaml
```

Once deployed, get the ALB DNS name from the stack outputs:

```bash
aws cloudformation describe-stacks \
  --stack-name alb-stack \
  --query 'Stacks[0].Outputs'
```

You can then access the site via:

```
http://<alb-dns-name>
```

> **Note:** The ALB will return a `503 Service Temporarily Unavailable` until EC2 instances are registered into the Target Group. This happens automatically once the Auto Scaling Group is deployed in the next step.

To check target health at any time:

```bash
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn>
```

---

### 8. Add the Launch Template

The Launch Template defines the blueprint for every EC2 instance the Auto Scaling Group will launch — AMI, instance type, security group, and the full WordPress UserData bootstrap script. It lives inside `autoscaling-alb.yaml` alongside the ALB.

Update the existing stack to add it:

```bash
aws cloudformation deploy \
  --stack-name alb-stack \
  --template-file templates/autoscaling-alb.yaml
```

To verify the Launch Template was created:

```bash
aws ec2 describe-launch-templates \
  --filters "Name=launch-template-name,Values=WordPressLaunchTemplate"
```

---

### 9. Add the Auto Scaling Group

The ASG uses the Launch Template to automatically launch and register EC2 instances into the ALB Target Group. It is configured with:

- **Min:** 2 instances
- **Desired:** 2 instances
- **Max:** 4 instances
- **Subnets:** both public subnets across 2 availability zones
- **Health checks:** EC2 + ELB (5-minute grace period for WordPress bootstrap)

Update the stack to add the ASG:

```bash
aws cloudformation deploy \
  --stack-name alb-stack \
  --template-file templates/autoscaling-alb.yaml
```

---

### 10. Verify Instances and Health Checks

Once the stack is deployed, confirm the ASG has launched its instances:

```bash
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names WordPressASG \
  --query 'AutoScalingGroups[0].Instances[*].[InstanceId,HealthStatus,LifecycleState,AvailabilityZone]' \
  --output table
```

Check that the instances pass the ALB health check. The UserData script takes ~5 minutes to complete, so wait before checking:

```bash
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn>
```

Both instances should show `"State": "healthy"`. You can get the Target Group ARN from the stack outputs:

```bash
aws cloudformation describe-stacks \
  --stack-name alb-stack \
  --query 'Stacks[0].Outputs'
```

Once both targets are healthy, the ALB DNS name will serve WordPress:

```
http://<alb-dns-name>
```

---

## Tearing Down the Stacks

Delete stacks in reverse order — dependents before dependencies. CloudFormation won't let you delete a stack whose exports are still being referenced by another.

```bash
# 1. Delete the ALB/ASG stack first
aws cloudformation delete-stack --stack-name alb-stack

# 2. Delete the WordPress web server stack
aws cloudformation delete-stack --stack-name wordpress-stack

# 3. Delete the network stack last
aws cloudformation delete-stack --stack-name network-stack
```

---

## Project Structure

```
project/
├── templates/
│   ├── network-security.yaml       # VPC, subnets, IGW, route tables, security groups
│   ├── wordpress-server.yaml       # EC2 instance with automated WordPress install
│   └── autoscaling-alb.yaml        # ALB, Target Group, Listener, Launch Template, ASG
├── screenshots/
├── docs/
│   └── deployment-guide.md         # Manual LAMP + WordPress installation guide
└── README.md
```

See `docs/deployment-guide.md` for the manual LAMP stack installation steps.
