# 🚀 AWS Auto Scaling Groups (ASG) - Learning Guide

*A beginner-friendly guide to understanding and creating AWS Auto Scaling Groups*

---

## 📚 Table of Contents

1. [What is Auto Scaling?](#what-is-auto-scaling)
2. [Why Do We Need It?](#why-do-we-need-it)
3. [Key Concepts](#key-concepts)
4. [Architecture Overview](#architecture-overview)
5. [Prerequisites](#prerequisites)
6. [Step-by-Step Tutorial](#step-by-step-tutorial)
7. [Testing Your ASG](#testing-your-asg)
8. [Troubleshooting](#troubleshooting)
9. [Best Practices](#best-practices)
10. [Quick Reference](#quick-reference)

---

## 🤔 What is Auto Scaling?

**Auto Scaling** is like having a smart robot that automatically adds or removes servers based on how busy your website/application is!

### Simple Analogy 🏪

Imagine you own a restaurant:
- **Slow day** (few customers): You keep 1-2 staff members → **Saves Money** 💰
- **Busy day** (many customers): You call more staff to help → **Better Service** ✅
- **Auto Scaling** does the same thing for your servers!

---

## 🎯 Why Do We Need It?

### Without Auto Scaling:
| Scenario | Problem |
|----------|---------|
| **Traffic Spikes** | Server crashes, users can't access |
| **Low Traffic** | Wasting money on unused servers |
| **Server Failure** | Website goes down completely |
| **Maintenance** | Downtime needed for updates |

### With Auto Scaling:
| Scenario | Solution |
|----------|----------|
| **Traffic Spikes** | Automatically adds more servers |
| **Low Traffic** | Removes extra servers to save cost |
| **Server Failure** | Automatically replaces failed servers |
| **Maintenance** | Rolling updates without downtime |

---

## 📖 Key Concepts

### 1. 🔧 Launch Template
> Think of it as a **"recipe"** or **"blueprint"** for creating new servers

Contains:
- Which AMI (image) to use
- What instance type (size)
- What security groups
- Startup scripts (user data)

### 2. 📦 Auto Scaling Group (ASG)
> The **manager** that controls your servers

Manages:
- **Minimum** servers (never go below this)
- **Maximum** servers (never go above this)
- **Desired** servers (target number)
- Which subnets to deploy in

### 3. 🏥 Health Checks
> The **doctor** that checks if servers are healthy

- **EC2 Health Checks**: Is the instance running?
- **ELB Health Checks**: Is the application responding?

### 4. 📈 Scaling Policies
> The **rules** that decide when to scale

Types:
- **Manual**: You control it yourself
- **Target Tracking**: Scale based on metric (like CPU %)
- **Scheduled**: Scale at specific times
- **Step**: Scale in steps based on conditions

---

## 🏗️ Architecture Overview

```
                           ┌─────────────────┐
                           │   INTERNET      │
                           └────────┬────────┘
                                    │
                           ┌────────▼────────┐
                           │ Internet Gateway│
                           └────────┬────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
           ┌────────▼────┐  ┌───────▼──────┐  ┌─────▼─────┐
           │ Public Subnet│  │Public Subnet │  │Public Subnet│
           │   (AZ-a)    │  │   (AZ-b)    │  │  (AZ-c)   │
           └──────┬──────┘  └──────┬───────┘  └──────┬─────┘
                  │                │                │
           ┌──────▼──────┐  ┌──────▼───────┐  ┌─────▼─────┐
           │   ALB/NLB   │  │   ALB/NLB   │  │  ALB/NLB  │
           └──────┬──────┘  └──────┬───────┘  └──────┬─────┘
                  │                │                │
        ┌─────────┼────────────────┼────────────────┼─────────┐
        │         │                │                │         │
   ┌────▼───┐ ┌───▼────┐     ┌─────▼─────┐    ┌─────▼────┐
   │Server 1│ │Server 2│     │ Server 3  │    │Server 4  │
   │(EC2)   │ │(EC2)   │     │  (EC2)    │    │  (EC2)   │
   └────┬───┘ └───┬────┘     └─────┬─────┘    └─────┬────┘
        │         │                │                │
        └─────────┼────────────────┼────────────────┘
                  ▼
       ┌──────────────────────┐
       │  AUTO SCALING GROUP  │
       │  ┌────────────────┐  │
       │  │ Min: 1         │  │
       │  │ Desired: 2    │  │
       │  │ Max: 4        │  │
       │  └────────────────┘  │
       └──────────────────────┘
```

---

## ✅ Prerequisites

Before starting, make sure you have:

- [ ] AWS Account (free tier works!)
- [ ] EC2 Key Pair (for SSH access)
- [ ] VPC with public and private subnets
- [ ] Security Group (with HTTP/HTTPS access)
- [ ] (Optional) Application Load Balancer already set up

> 💡 **Don't have these?** Check these guides first:
> - [VPC Setup Guide](01.Aws_VPC.md)
> - [ALB Setup Guide](02.aws_ALB.md)

---

## 📝 Step-by-Step Tutorial

### Step 1: Create Your Custom AMI (Server Image)

Think of AMI as a "snapshot" of your configured server.

#### 1.1 Launch a Base EC2 Instance

1. Go to **EC2 Dashboard** → **Launch Instance**
2. Fill in:
   - **Name**: `template-instance`
   - **AMI**: Amazon Linux 2
   - **Instance Type**: t2.micro
   - **Key Pair**: Your key pair
   - **VPC**: Your custom VPC
   - **Subnet**: Public subnet
   - **Security Group**: Web security group

#### 1.2 Install Your Application

```bash
# Connect to your server
ssh -i your-key.pem ec2-user@<public-ip>

# Install Apache web server
sudo yum update -y
sudo yum install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd

# Create a test page
echo "<h1>My Web Server</h1>" | sudo tee /var/www/html/index.html

# Test it
curl localhost
```

#### 1.3 Create AMI

1. Select your instance in EC2 Dashboard
2. **Actions** → **Image** → **Create Image**
3. Set:
   - **Image name**: `my-web-server-ami`
   - **Image description**: "Web server for ASG"
   - **No reboot**: ✅ Checked
4. Click **Create Image**
5. ⚠️ **Wait** until status shows "Available"

---

### Step 2: Create Launch Template

This is the "blueprint" for your ASG instances.

#### 2.1 Navigate to Launch Templates

**EC2 Dashboard** → **Launch Templates** → **Create launch template**

#### 2.2 Configure Basic Settings

| Field | Value |
|-------|-------|
| Launch template name | `my-asg-template` |
| Version description | "Version 1 - Web Server" |
| Provide guidance | Check "Provide guidance..." |

#### 2.3 Configure Instance Details

- **AMI**: Select `my-web-server-ami`
- **Instance Type**: t2.micro
- **Key Pair**: Your key pair
- **VPC**: Your custom VPC
- **Security Groups**: Your web security group
- **Subnet**: Leave empty (will specify in ASG)

#### 2.4 Add User Data (Startup Script)

This script runs when a new instance starts:

```bash
#!/bin/bash
# Auto Scaling Group Bootstrap Script

# Update and install Apache
yum update -y
yum install httpd -y

# Start Apache
systemctl start httpd
systemctl enable httpd

# Get instance info from AWS metadata
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s http://169.254.169.254/latest/meta-data/availability-zone)

# Create dynamic webpage
cat > /var/www/html/index.html <<EOF
<html>
<head><title>ASG Server</title></head>
<body>
<h1>Hello from Auto Scaling!</h1>
<p><strong>Instance ID:</strong> $INSTANCE_ID</p>
<p><strong>Availability Zone:</strong> $AZ</p>
<p><strong>Server Time:</strong> $(date)</p>
</body>
</html>
EOF
```

#### 2.5 Create the Template

Click **Create launch template** ✅

---

### Step 3: Create Auto Scaling Group

#### 3.1 Start Creating ASG

**EC2 Dashboard** → **Auto Scaling Groups** → **Create Auto Scaling group**

#### 3.2 Choose Launch Template

| Field | Value |
|-------|-------|
| Auto Scaling group name | `my-asg` |
| Launch template | `my-asg-template` |
| Version | Latest |

Click **Next**

#### 3.3 Configure Network

| Field | Value |
|-------|-------|
| VPC | Your custom VPC |
| Subnets | Select all private subnets |

✅ Select 3 private subnets in different Availability Zones for high availability

Click **Next**

#### 3.4 Configure Load Balancing (Recommended)

| Field | Value |
|-------|-------|
| Attach to existing load balancer | ✅ |
| Attach to target group | Select your ALB target group |
| Health check type | ELB |
| Health check grace period | 300 seconds |

Click **Next**

#### 3.5 Configure Group Size

| Field | Value | Why? |
|-------|-------|------|
| Desired capacity | 2 | Start with 2 servers |
| Minimum capacity | 1 | Never less than 1 |
| Maximum capacity | 4 | Never more than 4 |

> 💡 **Tip**: Start conservative, adjust later!

Click **Next**

#### 3.6 Configure Scaling Policies

**Option A: Manual Scaling**
- Select "No scaling policies"
- Good for learning/testing

**Option B: Automatic Scaling (Recommended)**

1. Select "Target tracking scaling policy"
2. Configure:
   - **Metric type**: Average CPU utilization
   - **Target value**: 70%
3. Click **Add policy**

> This means: "Add more servers when CPU usage is above 70%"

Click **Next**

#### 3.7 Add Notifications (Optional)

1. **Add notification**
2. **Topic**: Create new topic
3. **Topic name**: `asg-notifications`
4. **Events**: Select:
   - instance-launch
   - instance-terminate
   - instance-launch-failure
5. **Email**: Your email
6. Click **Next**

#### 3.8 Add Tags

1. **Add tag**
   - Key: `Name`
   - Value: `ASG-Instance`
   - Propagate: ✅ Checked

Click **Next**

#### 3.9 Review and Create

1. Review all settings
2. Click **Create Auto Scaling group** ✅

---

## 🧪 Testing Your ASG

### Test 1: Check Instances Are Running

1. Go to **Auto Scaling Groups**
2. Select `my-asg`
3. Check **Instances** tab
4. You should see 2 instances:
   - Lifecycle: "InService"
   - Health: "Healthy"

### Test 2: Manual Scale-Out

1. Select your ASG
2. Click **Edit**
3. Change **Desired capacity** from 2 → 3
4. Click **Save**
5. Wait 2-3 minutes
6. Check **Instances** tab - a new instance should appear! 🎉

### Test 3: Manual Scale-In

1. Edit ASG
2. Change **Desired capacity** from 3 → 2
3. Click **Save**
4. One instance will terminate gracefully

### Test 4: Automatic Scaling (CPU Test)

1. SSH into one of your instances
2. Install stress tool:
   ```bash
   sudo yum install stress -y
   ```
3. Run stress test:
   ```bash
   stress --cpu 80 --timeout 300 &
   ```
4. Watch your ASG - a new instance should launch when CPU > 70%
5. Stop the stress test - instance should terminate

### Test 5: Load Balancer Test

1. Copy your ALB DNS name
2. Open in browser multiple times
3. Each refresh should show different **Instance ID**!

---

## 🔧 Troubleshooting

### ❌ Instances Not Passing Health Checks

**Solutions:**
1. Check security group allows traffic from ALB
2. Verify health check path returns 200 OK
3. Check user data script is working

### ❌ ASG Not Scaling

**Solutions:**
1. Verify scaling policies are configured
2. Check CloudWatch alarms are active
3. Verify minimum capacity allows scaling

### ❌ Instance Launch Failures

**Solutions:**
1. Check AWS account instance limits
2. Verify key pair exists
3. Check VPC has available IPs
4. Review launch template for errors

---

## ⭐ Best Practices

### Security
- [ ] Always use custom AMIs with latest patches
- [ ] Use IAM roles instead of embedding credentials
- [ ] Restrict security groups to minimum required
- [ ] Enable termination protection

### Scaling
- [ ] Start with conservative min/max values
- [ ] Use multiple Availability Zones (3 AZs recommended)
- [ ] Configure appropriate health check grace period
- [ ] Test scaling policies before production

### Cost Optimization
- [ ] Use Reserved Instances for baseline capacity
- [ ] Use Spot Instances for fault-tolerant workloads
- [ ] Set appropriate CloudWatch alarm periods
- [ ] Always delete resources when done!

### High Availability
- [ ] Deploy across 3+ Availability Zones
- [ ] Use ELB health checks
- [ ] Configure appropriate cooldown periods

---

## 📋 Quick Reference

### AWS CLI Commands

```bash
# Create ASG
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name my-asg \
  --launch-template LaunchTemplateId=lt-xxx \
  --min-size 1 \
  --max-size 4 \
  --desired-capacity 2 \
  --vpc-zone-identifier "subnet-1,subnet-2"

# Update ASG capacity
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name my-asg \
  --desired-capacity 3

# Delete ASG
aws autoscaling delete-auto-scaling-group \
  --auto-scaling-group-name my-asg

# Check ASG status
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names my-asg
```

### Key Terms Glossary

| Term | Definition |
|------|------------|
| **AMI** | Amazon Machine Image - a template for your server |
| **ASG** | Auto Scaling Group - manages your server fleet |
| **ELB** | Elastic Load Balancer - distributes traffic |
| **AZ** | Availability Zone - isolated data center |
| **VPC** | Virtual Private Cloud - your virtual network |
| **Grace Period** | Time to wait before health checks start |
| **Cooldown Period** | Time between scaling activities |

---

## 🧹 Cleanup (IMPORTANT!)

Don't forget to delete resources to avoid charges:

### Step-by-Step Cleanup

1. **Delete Auto Scaling Group**
   - EC2 → Auto Scaling Groups → Delete

2. **Delete Launch Template**
   - EC2 → Launch Templates → Delete

3. **Delete Target Groups**
   - EC2 → Target Groups → Delete

4. **Delete Load Balancer**
   - EC2 → Load Balancers → Delete

5. **Terminate Instances**
   - EC2 → Instances → Terminate

6. **Delete AMI**
   - AMIs → Deregister image → Delete snapshot

---

## 🎓 What You Learned

| Component | Purpose |
|-----------|---------|
| **Custom AMI** | Pre-configured server image with your app |
| **Launch Template** | Blueprint for creating new instances |
| **Auto Scaling Group** | Manages server fleet automatically |
| **Load Balancer** | Distributes traffic to healthy servers |
| **Scaling Policies** | Rules for when to add/remove servers |
| **Health Checks** | Monitors server health |

---

## 🔗 Related Guides

- [AWS VPC Setup Guide](01.Aws_VPC.md) - Learn about networking
- [AWS ALB Setup Guide](02.aws_ALB.md) - Learn about load balancing
- [EC2 and EBS Guide](06.AWS_EC2_EBS_Snapshots_ALB-and-NLB.md) - Learn about EC2

---

## 🎉 Congratulations!

You've completed the AWS Auto Scaling Groups tutorial!

### What You Can Now Do:
✅ Create custom AMIs  
✅ Configure Launch Templates  
✅ Set up Auto Scaling Groups  
✅ Configure scaling policies  
✅ Test auto scaling behavior  
✅ Troubleshoot common issues  

---

## 📚 Additional Resources

- [AWS Auto Scaling Documentation](https://docs.aws.amazon.com/autoscaling/)
- [AWS EC2 Auto Scaling User Guide](https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-aws-autoscaling.html)
- [AWS Launch Templates](https://docs.aws.amazon.com/autoscaling/ec2/userguide/LaunchTemplates.html)

---

<div align="center">

**Happy Learning & Happy DevOps!** 🚀

*Made with ❤️ for learning purposes*

</div>

