# WordPress on AWS with Monitoring & DEV Schedule (CloudFormation)

CloudFormation templates to deploy a single WordPress EC2 instance **in a private subnet** behind an **internet-facing ALB**, with **CloudWatch logs/metrics**, **VPC Flow Logs**, and a **DEV working-hours schedule** (start 09:00 / stop 18:05 America/Guayaquil, Mon–Fri) via **EventBridge + Lambda**.

## Architecture

- **VPC /16** with **2 public subnets** (for ALB across 2 AZs) and **1 private subnet** (EC2).
- **Internet Gateway** for inbound via ALB; **NAT Gateway** for EC2 outbound.
- **ALB → Target Group (HTTP:80)** with `HealthCheckPath: /health.html` (HTTP 200).
- **EC2 (Amazon Linux 2023/2)** installs Apache, PHP, MariaDB, WordPress via UserData; **CloudWatch Agent** for metrics and log shipping.
- **CloudWatch**: log groups for network and app, metrics, alarms (EC2 CPU high, ALB 5xx).
- **EventBridge + Lambda**: start at **09:00** and stop at **18:05** (Mon–Fri, America/Guayaquil = UTC−5; cron evaluated in UTC).

```
.
├── 01-network.yaml        # VPC, subnets, IGW, NAT, VPC Flow Logs → CloudWatch
├── 02-wordpress.yaml      # EC2 (private), ALB (public), TG, SGs, IAM, logs, alarms
└── 03-schedule-dev.yaml   # EventBridge (UTC cron) + Lambda start/stop
```

## Prerequisites

- AWS account with permissions for VPC, EC2, ELBv2, IAM, CloudWatch, EventBridge, Lambda, SNS.
- Region: examples use **us-east-1**.
- AWS CLI configured.
- Use `CAPABILITY_NAMED_IAM` for the IAM roles/policies created by the templates.

## Deploy

All commands assume **us-east-1**. Run from **AWS CloudShell** or your terminal.

### 1) Network stack

Creates VPC, two public subnets (AZ A/B), one private subnet, IGW, optional NAT, a log group, and **VPC Flow Logs** to CloudWatch. Exports are used by the app stack.

```bash
aws cloudformation deploy   --stack-name wordpress-network-dev   --template-file 01-network.yaml   --parameter-overrides       ProjectName=wp-course-project       Environment=dev       CreateNatGateway=true       VpcCidr=10.1.0.0/16       PublicSubnetCidr=10.1.0.0/24       PublicSubnetCidrB=10.1.2.0/24       PrivateSubnetCidr=10.1.1.0/24   --capabilities CAPABILITY_NAMED_IAM   --region us-east-1
```

**Exports used later:** `VpcId`, `PublicSubnetId`, `PublicSubnetIdB`, `PrivateSubnetId`.

### 2) Application stack (WordPress + ALB)

Launches the EC2 in the private subnet, creates the ALB in the two public subnets, Target Group with `/health.html`, SGs, IAM (SSM + CloudWatch Agent), app log group, and CloudWatch alarms.

```bash
aws cloudformation deploy   --stack-name wordpress-app-dev   --template-file 02-wordpress.yaml   --parameter-overrides       ProjectName=wp-course-project       Environment=dev       NetworkStackName=wordpress-network-dev       InstanceType=t3.small       AdminCidr=0.0.0.0/0       KeyName=   --capabilities CAPABILITY_NAMED_IAM   --region us-east-1
```

**Validate:**  
- EC2 Target Group → **Healthy**.  
- Open the `AlbDNSName` output in your browser (WordPress installer).  
- Use **SSM Session Manager** for ops (no public SSH required).

### 3) DEV schedule stack (start/stop)

EventBridge cron is evaluated in **UTC**. America/Guayaquil (UTC−5):  
09:00 local → **14:00 UTC**; 18:05 local → **23:05 UTC**.

```bash
INSTANCE_ID=$(aws cloudformation describe-stacks   --stack-name wordpress-app-dev   --query "Stacks[0].Outputs[?OutputKey=='InstanceId'].OutputValue"   --output text --region us-east-1)

aws cloudformation deploy   --stack-name wordpress-schedule-dev   --template-file 03-schedule-dev.yaml   --parameter-overrides       ProjectName=wp-course-project       Environment=dev       InstanceId=$INSTANCE_ID   --capabilities CAPABILITY_NAMED_IAM   --region us-east-1
```

**Manual test:**
```bash
FN=wp-course-project-dev-startstop

aws lambda invoke --function-name $FN --payload '{"action":"stop"}'   --cli-binary-format raw-in-base64-out /dev/stdout --region us-east-1 ; echo
aws ec2 wait instance-stopped --instance-ids $INSTANCE_ID --region us-east-1

aws lambda invoke --function-name $FN --payload '{"action":"start"}'   --cli-binary-format raw-in-base64-out /dev/stdout --region us-east-1 ; echo
aws ec2 wait instance-running --instance-ids $INSTANCE_ID --region us-east-1
```

## Parameters (summary)

### `01-network.yaml`
| Name | Type | Default | Description |
|---|---|---|---|
| `ProjectName` | String | `wp-course-project` | Tagging prefix. |
| `Environment` | String | `dev` | `dev` or `prod`. |
| `CreateNatGateway` | String | `true` | Create NAT for private outbound. |
| `VpcCidr` | String | `10.1.0.0/16` | VPC CIDR. |
| `PublicSubnetCidr` | String | `10.1.0.0/24` | Public Subnet A. |
| `PublicSubnetCidrB` | String | `10.1.2.0/24` | Public Subnet B (second AZ). |
| `PrivateSubnetCidr` | String | `10.1.1.0/24` | Private Subnet. |

**Key outputs:** `VpcId`, `PublicSubnetId`, `PublicSubnetIdB`, `PrivateSubnetId`.

### `02-wordpress.yaml`
| Name | Type | Default | Description |
|---|---|---|---|
| `ProjectName` | String | `wp-course-project` | Tagging prefix. |
| `Environment` | String | `dev` | `dev` or `prod`. |
| `NetworkStackName` | String | `wordpress-network-dev` | Export source. |
| `InstanceType` | String | `t3.small` | EC2 size. |
| `AmiId` | AWS::EC2::Image::Id | region-specific | Amazon Linux 2023/2 AMI. |
| `AdminCidr` | String | `0.0.0.0/0` | SSH ingress (limit to your `/32`). |
| `KeyName` | String | `""` | Optional EC2 KeyPair. |
| `AppLogRetentionDays` | Number | `30` | App log retention. |

**Key outputs:** `AlbDNSName`, `InstanceId`, `AppSecurityGroupId`, `AlbSecurityGroupId`, `AppLogGroupName`.

### `03-schedule-dev.yaml`
| Name | Type | Default | Description |
|---|---|---|---|
| `ProjectName` | String | `wp-course-project` | Tagging prefix. |
| `Environment` | String | `dev` | `dev` or `prod`. |
| `InstanceId` | String | — | EC2 to start/stop. |
| `LogRetentionDays` | Number | `14` | Lambda log retention. |

**Key outputs:** `FunctionName`, `StartRuleName`, `StopRuleName`.

## Monitoring

- **CloudWatch Logs**  
  - Network (VPC Flow Logs) log group (created by network stack).  
  - App log group: Apache access/error and system messages.  
  - Lambda log group: start/stop invocations.
- **Metrics** via CloudWatch Agent: CPU, memory, disk (1-minute).  
- **Alarms**: EC2 CPUUtilization high; ALB HTTP 5xx count.

## Troubleshooting

- **ALB requires two public subnets** in different AZs; ensure both are exported by the network stack.  
- **Target Group unhealthy / 502**: verify Apache is running via **SSM Session Manager**, confirm `/health.html` returns HTTP 200, and set TG `HealthCheckPath=/health.html` with matcher `200`.  
- **Amazon Linux 2023**: MariaDB package is `mariadb105-server`; UserData handles this.

## Security & Cost

- EC2 in a **private** subnet (no public IP); outbound via **NAT**.  
- App SG: `80/TCP` only from ALB SG; `22/TCP` from `AdminCidr` (restrict to `/32`).  
- Set CloudWatch log retention to meet policy.  
- NAT Gateway and ALB incur cost; DEV schedule stops the instance outside business hours.

## Cleanup

1. Delete stack `wordpress-schedule-dev`.  
2. Delete stack `wordpress-app-dev`.  
3. Delete stack `wordpress-network-dev`.

## License

MIT
