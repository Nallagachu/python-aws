
# 🚀 Automating EC2 Start/Stop Scheduling with AWS Lambda, EventBridge & Terraform

## 📖 Project Overview

This project automates the **start and stop of EC2 instances** at scheduled times using:

* **Terraform** → Infrastructure as Code (IaC) to provision resources (IAM roles, policies, S3 backend, Lambda, EventBridge).
* **AWS Lambda (Python)** → Executes the start/stop logic for EC2 instances.
* **Amazon EventBridge** → Triggers Lambda on a defined schedule (cron/interval).
* **CloudWatch Logs** → Captures Lambda logs for monitoring & debugging.

Additionally, the project demonstrates how Lambda can process **security vulnerability reports** from **S3 buckets** using Python & Pandas, and automatically create/update **Jira tickets**.

---

## ⚙️ Workflow Summary

1. **Terraform Setup**

   * Configure backend (`provider.tf`) with S3 bucket & state file.
   * Create IAM role (`iam.tf`) with policies (`permissions.tf`) for Lambda.

2. **Lambda Function**

   * Python script (`ec2_autostart.py`) to **start/stop EC2 instances** based on tags.
   * Handles **pagination** in AWS responses.
   * Packaged into `.zip` and deployed with Terraform (`lambda.tf`).

3. **EventBridge Rule**

   * Defines **cron expression** for execution.
   * Triggers Lambda at scheduled times (UTC → convert from IST if required).

4. **Monitoring & Debugging**

   * Logs available in **CloudWatch Log Groups**.
   * Helps troubleshoot (e.g., missing tags, wrong schedule, permissions).

5. **Extended Use Case (Optional)**

   * Lambda triggered on **S3 object upload**.
   * Processes vulnerability reports with **Pandas**.
   * Compares **existing vs new vulnerabilities** and updates **Jira tickets** automatically.

---

## 📂 Project Structure

```
aws-ec2-scheduler/
│── provider.tf           # AWS provider & backend config
│── iam.tf                # IAM roles for Lambda
│── permissions.tf        # IAM policies for Lambda
│── lambda.tf             # Lambda function definition
│── eventbridge.tf        # EventBridge rule for scheduling
│── scripts/
│   └── ec2_autostart.py  # Python script for EC2 start/stop
│   └── vuln_process.py   # (Optional) Vulnerability report processor
│── README.md             # Documentation
```

---

## 🔑 IAM Roles & Policies

Lambda requires permissions to:

* Write to **CloudWatch Logs**.
* **Describe, Start, Stop EC2 instances**.

Example Terraform policy snippet:

```hcl
resource "aws_iam_role" "lambda_exec" {
  name = "ec2-scheduler-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy" "lambda_policy" {
  name = "lambda-ec2-policy"
  role = aws_iam_role.lambda_exec.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"]
        Resource = "arn:aws:logs:*:*:*"
      },
      {
        Effect   = "Allow"
        Action   = ["ec2:StartInstances", "ec2:StopInstances", "ec2:DescribeInstances"]
        Resource = "*"
      }
    ]
  })
}
```

---

## 🐍 Lambda Function – EC2 Start/Stop (`ec2_autostart.py`)

```python
import boto3
import os

ec2 = boto3.client('ec2')

def lambda_handler(event, context):
    action = event.get("action", "stop")  # default action is stop
    tag_key = os.getenv("TAG_KEY", "Environment")
    tag_value = os.getenv("TAG_VALUE", "Dev")

    filters = [{'Name': f'tag:{tag_key}', 'Values': [tag_value]}]

    response = ec2.describe_instances(Filters=filters)

    instance_ids = [
        instance["InstanceId"]
        for reservation in response["Reservations"]
        for instance in reservation["Instances"]
        if instance["State"]["Name"] in ["running", "stopped"]
    ]

    if not instance_ids:
        print("No instances found with given tags")
        return

    if action == "start":
        ec2.start_instances(InstanceIds=instance_ids)
        print(f"Started instances: {instance_ids}")
    else:
        ec2.stop_instances(InstanceIds=instance_ids)
        print(f"Stopped instances: {instance_ids}")
```

✅ Uses **tags** to filter instances.
✅ Supports **start** or **stop** via input event.
✅ Logs to **CloudWatch**.

---

## 📅 EventBridge Scheduling

Terraform snippet for EventBridge rule:

```hcl
resource "aws_cloudwatch_event_rule" "schedule" {
  name                = "ec2-scheduler-rule"
  schedule_expression = "cron(30 2 * * ? *)" # Example: 8:00 AM IST (2:30 UTC)
}

resource "aws_cloudwatch_event_target" "lambda_target" {
  rule      = aws_cloudwatch_event_rule.schedule.name
  target_id = "ec2-scheduler"
  arn       = aws_lambda_function.ec2_scheduler.arn
}
```

---

## ⚡ Deployment Steps

1. **Initialize Terraform**

   ```sh
   terraform init -upgrade
   ```

2. **Plan & Review**

   ```sh
   terraform plan
   ```

3. **Apply Changes**

   ```sh
   terraform apply -auto-approve
   ```

4. **Package Python Script**

   ```sh
   zip function.zip ec2_autostart.py
   ```

5. **Lambda Deploys** via Terraform (`lambda.tf`).

---

## 🐛 Debugging & Fixes

* **Issue:** Lambda executes but instances don’t change state
  ✅ Check EC2 **tags** match filter.
  ✅ Check EventBridge **time zone** (use UTC).

* **Issue:** Timeout
  ✅ Increase Lambda timeout from default 3s → 30s.

* **Issue:** No logs
  ✅ Verify IAM role has CloudWatch Logs permissions.

---

## 📊 Extended Use Case – Vulnerability Reports

* New reports uploaded to **S3** trigger Lambda.
* Python (`vuln_process.py`) processes reports with Pandas:

  * Segregates **new vs existing vulnerabilities**.
  * Maintains **Excel sheets** for tracking.
  * Updates **Jira tickets** automatically using APIs.

---

## ✅ Best Practices

* Always use **tags** for instance selection.
* Keep schedules in **UTC**, then adjust from IST.
* Use **Terraform backend (S3 + DynamoDB)** for safe state management.
* Keep Lambda **lightweight** (only required packages).
* Monitor execution in **CloudWatch Logs** after every deployment.

---

## 🚀 Outcome

* Fully automated **EC2 instance scheduling**.
* Clear visibility with **CloudWatch Logs**.
* Scalable design (add more rules for multiple environments).
* Optional **security automation** with S3 + Jira integration.

Absolutely! Let’s structure **clear workflows** for your project. I’ll break them into **two main workflows**:

1. **EC2 Start/Stop Scheduling**
2. **Vulnerability Report Automation (Optional/Advanced)**

I’ll make them **step-by-step, visualizable, and actionable** for your README or GitHub wiki.

---

## 1️⃣ Workflow: EC2 Start/Stop Scheduling

**Objective:** Automatically start and stop EC2 instances based on tags and schedule using Lambda + EventBridge + Terraform.

### Step-by-Step Workflow:

1. **Terraform Setup**

   * Create `provider.tf` → AWS provider + S3 backend for state.
   * Create `iam.tf` → IAM role for Lambda.
   * Create `permissions.tf` → Policies: EC2 start/stop/describe + CloudWatch Logs.

2. **Prepare Lambda Python Script**

   * `ec2_autostart.py` filters instances by **tags**.
   * Implements **start/stop logic**.
   * Handles AWS **pagination** (`describe_instances` may return multiple pages).

3. **Package Lambda**

   * Zip Python script: `zip function.zip ec2_autostart.py`.
   * Store zip in Terraform (`lambda.tf`) for deployment.

4. **EventBridge Scheduling**

   * Create **rule** with cron schedule (convert IST → UTC).
   * Target Lambda function.

5. **Terraform Deployment**

   * `terraform init -upgrade`
   * `terraform plan`
   * `terraform apply -auto-approve`

6. **Monitoring & Logging**

   * Lambda execution logs → CloudWatch Log Groups.
   * Check logs to ensure instances are **starting/stopping** as expected.

7. **Maintenance**

   * Update tags if new EC2 instances are added.
   * Adjust cron schedule if needed.
   * Destroy resources if no longer required: `terraform destroy -auto-approve`.

**Outcome:** EC2 instances automatically start/stop at scheduled times based on tags.

---

 2️⃣ Workflow: Vulnerability Report Automation 

**Objective:** Automatically process vulnerability reports uploaded to S3, update Excel sheets, and create/update Jira tickets.

### Step-by-Step Workflow:

1. **S3 Setup**

   * Store vulnerability reports in a dedicated bucket.
   * Configure Lambda **trigger** on S3 upload.

2. **Lambda Python Script**

   * `vuln_process.py` reads CSV/Excel using **Pandas**.
   * Segregates **new vs existing vulnerabilities**.
   * Updates two separate Excel sheets:

     * Existing vulnerabilities
     * New vulnerabilities

3. **Jira Integration**

   * For **new vulnerabilities** → Create Jira tickets automatically via API.
   * For **existing vulnerabilities** → Add **comments or updates** to existing tickets.

4. **Deployment & Trigger**

   * Package Lambda script as `.zip`.
   * Deploy via Terraform.
   * EventBridge automatically triggers Lambda if needed (optional).

5. **Monitoring & Logging**

   * CloudWatch logs capture processing steps.
   * Validate Excel sheets and Jira ticket creation.
