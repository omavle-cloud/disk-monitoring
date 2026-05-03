# Disk Monitoring Solution — AWS

Ansible-driven disk utilisation monitoring across multiple AWS accounts.
Detects low disk space early and minimises risk of downtime using
CloudWatch metrics, alarms, and SNS notifications.

---

## Architecture

           +-----------------------------+
           |     Ansible Control Node    |
           +-------------+---------------+
                         |
                AssumeRole (IAM)
                         |
        +----------------+----------------+
        |                                 |
+---------------+                +----------------+
| AWS Account A |                | AWS Account B  |
+---------------+                +----------------+
        |                                 |
   EC2 Instances                     EC2 Instances
        |                                 |
  CloudWatch Agent                CloudWatch Agent
        |                                 |
        +-------------+-------------------+
                      |
               CloudWatch Metrics
                      |
         +------------+-------------+
         |                          |
   CloudWatch Alarms           Dashboards
         |
        SNS Notifications

---

## Prerequisites

| Requirement | Notes |
|---|---|
| Ansible >= 2.14 | With `amazon.aws` collection |
| Python 3 | Must be pre-installed on all target EC2 instances |
| boto3 / botocore | Required on the Ansible control node |
| AWS CLI v2 | Configured with valid credentials |
| SSH key pair | For connecting to EC2 instances |

Install Ansible dependencies:
```bash
pip install boto3 botocore
ansible-galaxy collection install amazon.aws
```

---

## Access Management

The Ansible control node connects to EC2 instances using SSH with a key pair. For multi-account setups, the control node IAM role uses `sts:AssumeRole` to access instances in other accounts — see `inventory/account1_aws_ec2.yml.disabled` for the SSM-based multi-account configuration.

No credentials are stored in any file. The AWS account ID is passed at runtime:
```bash
ansible-playbook playbook/alarms.yml -e "aws_account_id=YOUR_ACCOUNT_ID"
```

Each workload account requires a cross-account IAM role with these minimum permissions:

| Permission | Purpose |
|---|---|
| `ec2:DescribeInstances` | Dynamic inventory discovery |
| `cloudwatch:PutMetricData` | Publish disk metrics |
| `cloudwatch:PutMetricAlarm` | Create alarms |
| `ssm:StartSession` | SSM-based connection (when enabled) |

---

## VM Discovery and Enrollment

EC2 instances are discovered automatically using the `aws_ec2` dynamic inventory plugin. It queries AWS at runtime and returns all running EC2 instances — no static host files needed.

To target a specific region or group:
```bash
# All instances
ansible-playbook -i inventory/aws_ec2.yml playbook/install_agent.yml

# Specific region
ansible-playbook -i inventory/aws_ec2.yml playbook/install_agent.yml --limit region_us_east_1
```

### Single account — SSH (current)
Active `inventory/aws_ec2.yml` connects via SSH using the instance public IP. Suitable for single-account use.

### Multi-account — SSM (production)
`account1_aws_ec2.yml.disabled` and `account2_aws_ec2.yml.disabled` show the multi-account SSM-based setup. To enable for an account:
1. Remove the `.disabled` extension from the file
2. Update `aws_profile` and `ansible_aws_ssm_bucket_name` values
3. Ensure `AmazonSSMManagedInstanceCore` policy is attached to the EC2 instance IAM role

---

## Data Collection and Aggregation

The CloudWatch Agent is installed on every VM by `install_agent.yml`. It pushes the following metrics to the `DiskMonitoring` CloudWatch namespace every 60 seconds:

| Metric | Description |
|---|---|
| `disk_used_percent` | Percentage of disk used per mount point |
| `disk_used` | Bytes used |
| `disk_total` | Total disk size |
| `disk_free` | Bytes free |

Pseudo-filesystems (`tmpfs`, `devtmpfs`, `squashfs`, `overlay`) are excluded automatically. All mount points are monitored via `"resources": ["*"]` in the agent config.

---

## Alerting

`playbook/alarms.yml` creates a CloudWatch alarm for every instance and every mount point that has reported metrics.

| Setting | Value |
|---|---|
| Threshold | 80% disk used |
| Period | 5 minutes |
| Evaluation periods | 2 (fires after 10 consecutive minutes) |
| Missing data | Treated as not breaching |
| Recovery alert | SNS notification sent on recovery too |

---

## Quick Start

```bash
# 1. Clone the repository
git clone https://github.com/your-org/disk-monitoring.git
cd disk-monitoring

# 2. Install CloudWatch Agent on all EC2
ansible-playbook -i inventory/aws_ec2.yml playbook/install_agent.yml --private-key ~/.ssh/your-key.pem

# 3. Create CloudWatch alarms
ansible-playbook -i inventory/aws_ec2.yml playbook/alarms.yml -e "aws_account_id=YOUR_ACCOUNT_ID"
```

---

## Scalability

Adding a new AWS account requires two steps:

1. Copy and enable a new inventory file:
```bash
cp inventory/account1_aws_ec2.yml.disabled inventory/account3_aws_ec2.yml
# Edit the file — update aws_profile and bucket name
```

2. Ensure the cross-account IAM role exists in the new account with the permissions listed in the Access Management section above.

The dynamic inventory picks up all instances in the new account automatically on the next run. No other changes are required.