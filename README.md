**load-balanced-autoscaling-webservers** — Terraform configuration that provisions a production-grade, self-healing web tier on AWS: an Application Load Balancer fronting an Auto Scaling Group of Apache HTTP servers that scale between 2 and 10 instances based on demand.

---

## Project at a Glance

| | |
|---|---|
| **Tools Used** | Terraform, AWS ALB, AWS Auto Scaling Group, AWS Launch Template, AWS SSM Parameter Store |
| **Platform** | Amazon Web Services (us-east-1) |
| **Languages** | HCL (Terraform), Bash (user data) |
| **What It Does** | Deploys a horizontally scalable Apache web tier behind an Application Load Balancer, with ELB-based health checks, a minimum of 2 running instances, and automatic scale-out to 10 instances — serving a static web application fetched at boot time |

---

## The Problem This Project Solves

A single EC2 instance running a web server is a single point of failure. When that instance experiences a hardware fault, exhausts its resources under load, or requires patching, the application goes down. Manual remediation takes time, creates operational risk, and does not scale. Organizations that rely on fixed-capacity infrastructure must choose between over-provisioning (wasting money during low-traffic periods) or under-provisioning (risking outages during traffic spikes).

This project solves that problem with a fully declarative, elastic web tier. An AWS Auto Scaling Group maintains a minimum of two EC2 instances at all times — providing redundancy by default — and can expand to ten instances automatically when load increases. Every instance is launched from the same immutable Launch Template: a `t2.micro` Amazon Linux 2 instance whose user data script installs Apache HTTPD, downloads a web application from GitHub at boot time, and starts the web server. The instances register themselves with an ALB Target Group, and the ALB's ELB health check continuously monitors each instance, removing unhealthy members from rotation and replacing them.

The Application Load Balancer sits in front of the Auto Scaling Group and distributes incoming HTTP traffic across all healthy instances in the default VPC's subnets. Because the ALB's DNS name is exported as a Terraform output, the application URL is available immediately after `terraform apply` — no manual lookup required. From an infrastructure-as-code perspective, the use of `create_before_destroy` lifecycle hooks on both the Launch Template and Target Group ensures zero-downtime updates: replacement resources are created before the originals are removed.

---

## Architecture

```
Internet
   |
   | HTTP :80
   v
+--------------------------------------+
|  Application Load Balancer (ALB)     |
|  - Listener: TCP 80                  |
|  - Default action: 404 fixed response|
|  - Rule: forward /* to Target Group  |
|  Security Group: allow TCP 80 in     |
+------------------+-------------------+
                   |
                   | Health check: GET / -> HTTP 200
                   | interval: 15s, timeout: 3s
                   | healthy threshold: 2
                   v
+--------------------------------------+
|  ALB Target Group                    |
|  - Protocol: HTTP, Port: 80          |
|  - Registered targets: ASG instances |
+------------------+-------------------+
                   |
                   v
+--------------------------------------+
|  Auto Scaling Group                  |
|  - Min: 2  Desired: 2  Max: 10       |
|  - Health check type: ELB            |
|  - Subnets: all default VPC subnets  |
+------------------+-------------------+
                   |
                   v
+--------------------------------------+
|  Launch Template                     |
|  - AMI: Amazon Linux 2 (SSM lookup)  |
|  - Instance type: t2.micro           |
|  - User data (bash):                 |
|    yum install httpd wget unzip      |
|    wget CSCC1030 app from GitHub     |
|    unzip + move to /var/www/html     |
|    service httpd start               |
|  Security Group: allow TCP 80, 22 in |
+--------------------------------------+
```

---

## Repository Structure

```
load-balanced-autoscaling-webservers/
├── main.tf        # All resources: Launch Template, ASG, ALB, Target Group,
│                  # Listener, Listener Rule, Security Groups, data sources
├── variables.tf   # Variables: server port, ALB name, security group names
└── outputs.tf     # Output: ALB DNS name (application URL)
```

---

## How It Works

Terraform uses `data "aws_vpc" "default"` and `data "aws_subnets" "default"` to deploy into the account's default VPC, making this configuration immediately usable without pre-created networking infrastructure. The AMI is resolved dynamically from AWS SSM Parameter Store (`/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2`), ensuring the Launch Template always references a current, patched image.

The `aws_launch_template` resource encodes the complete instance configuration as base64-encoded user data. When an instance launches, it runs a shell script that updates packages, installs Apache HTTPD, wget, and unzip, downloads the CSCC1030 web application archive from GitHub, extracts it into `/var/www/html`, and starts the HTTPD service. The `create_before_destroy` lifecycle rule ensures that when the Launch Template is updated (e.g., for a new application version), the ASG creates instances from the new template before terminating the old ones.

The `aws_autoscaling_group` references the Launch Template by ID and version (`$Latest`) and sets `health_check_type = "ELB"` — meaning the ALB, not just EC2 status checks, determines instance health. The ALB listener on port 80 returns a 404 fixed response by default, with a forwarding rule that matches all paths (`*`) and routes them to the target group. This pattern allows additional path-based routing rules to be added without modifying the listener.

---

## Walkthrough

**1. Initialize Terraform**
```bash
terraform init
```

**2. Review the execution plan**
```bash
terraform plan
```

**3. Deploy the full stack**
```bash
terraform apply
```

**4. Retrieve the application URL**
```bash
terraform output alb_dns_name
# Example: terraform-asg-example-1234567890.us-east-1.elb.amazonaws.com
```

**5. Verify the application is serving**
```bash
curl http://$(terraform output -raw alb_dns_name)
```
The ALB may take 1-2 minutes after `apply` completes for the health checks to pass and instances to enter the `InService` state.

**6. Observe the Auto Scaling Group**
```bash
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names terraform-asg \
  --query 'AutoScalingGroups[0].Instances[*].[InstanceId,HealthStatus,LifecycleState]' \
  --output table
```

**7. Destroy all resources**
```bash
terraform destroy
```

---

## How to Reproduce

**Prerequisites**
- Terraform >= 0.15.0
- AWS credentials configured with permissions for EC2, ELB, and Auto Scaling
- The default VPC must exist in the target region (it does in new AWS accounts by default)

```bash
# Clone the repository
git clone https://github.com/<your-username>/load-balanced-autoscaling-webservers.git
cd load-balanced-autoscaling-webservers

# Deploy
terraform init
terraform apply -auto-approve

# Get the load balancer URL
terraform output alb_dns_name

# Wait ~90 seconds for health checks, then test
curl http://$(terraform output -raw alb_dns_name)

# Clean up
terraform destroy -auto-approve
```

To customize the ALB or security group names to avoid conflicts in a shared AWS account:
```bash
terraform apply \
  -var='alb_name=my-custom-alb' \
  -var='instance_security_group_name=my-instance-sg' \
  -var='alb_security_group_name=my-alb-sg'
```

---

*Oluwaseyi Michael Falode · Cybersecurity & Cloud Security Engineer · Toronto, ON*
