---

# Assignment 5: Auto‑Tagging EC2 Instances on Launch Using AWS Lambda and Boto3

## 📌 Objective
Automate the tagging of EC2 instances as soon as they are launched, ensuring better resource tracking and management.  
This Lambda function automatically tags any newly launched EC2 instance with the **current date** and a **custom tag**.

---

## ⚙️ Setup Instructions

### 1. EC2 Setup
- Ensure you have permissions to launch EC2 instances in your AWS account.
- No special configuration is required on the EC2 side; the Lambda will handle tagging after launch.

---

### 2. IAM Role for Lambda
- Go to the **IAM dashboard**.
- Click **Roles → Create role**.
- **Trusted entity**: Select **AWS service → Lambda**.
- Attach policies:
  - `AWSLambdaBasicExecutionRole` → allows logging to CloudWatch.
  - `AmazonEC2FullAccess` (for simplicity).  
    ⚠️ Best practice: replace with a custom policy that only allows `ec2:CreateTags` and `ec2:DescribeInstances`.
- Name the role (e.g., `LambdaEC2AutoTaggingRole`).

📸 *Screenshot:* ![alt text](screenshots/iam-role.png)

---

### 3. Lambda Function (Python 3.x)
Create a new Lambda function in the **Lambda dashboard**:

```python
import boto3
from datetime import datetime

ec2 = boto3.client('ec2')

def lambda_handler(event, context):
    instance_id = event['detail']['instance-id']
    current_date = datetime.utcnow().strftime('%Y-%m-%d')
    
    ec2.create_tags(
        Resources=[instance_id],
        Tags=[
            {'Key': 'LaunchDate', 'Value': current_date},
            {'Key': 'Owner', 'Value': 'AutoTagLambda'}
        ]
    )
    
    print(f"Instance {instance_id} tagged with LaunchDate={current_date} and Owner=AutoTagLambda")
    
    return {
        "status": "Tagging complete",
        "instance_id": instance_id,
        "tags": {
            "LaunchDate": current_date,
            "Owner": "AutoTagLambda"
        }
    }
```

📸 *Screenshot:* ![alt text](screenshots/lambda-function.png)

---

### 4. EventBridge Rule (CloudWatch Events)

#### Step‑by‑Step in Console
1. **Open EventBridge** → search for *EventBridge* in the AWS Console.
2. Click **Rules → Create rule**.
3. Enter:
   - Name: `EC2AutoTagRule`
   - Event bus: `default`
   - Rule type: **AWS service events**

4. **Define Event Pattern**
   - Service: **EC2**
   - Event type: **EC2 Instance State-change Notification**
   - Switch to **JSON editor** and paste:
     ```json
     {
       "source": ["aws.ec2"],
       "detail-type": ["EC2 Instance State-change Notification"],
       "detail": {
         "state": ["running"]
       }
     }
     ```

📸 *Screenshot:* ![alt text](screenshots/eventbridge-event-pattern.png)

5. **Add Target**
   - Target: **Lambda function**
   - Select your Lambda (e.g., `AutoTagLambda`)
   - Leave permissions as default → **Create a new role for this resource**

📸 *Screenshot:* ![alt text](screenshots/eventbridge-target.png)

6. **Review and Create** → click **Create rule**.

---

## 🔄 Trigger Flow Diagram

```
+---------+        +-------------------+        +------------------+        +----------------+
|  EC2    | -----> | EventBridge Rule  | -----> | Lambda Function  | -----> | EC2 Tags Added |
| Launch  |        | (State=running)   |        | (AutoTag Script) |        | (LaunchDate,   |
| Instance|        |                   |        |                  |        | Owner)         |
+---------+        +-------------------+        +------------------+        +----------------+
```

---

### 5. Testing
1. Launch a new EC2 instance (or stop/start an existing one to trigger a new `running` event).
2. Wait until it enters the **running** state.
3. EventBridge will send the event to your Lambda.
4. Lambda extracts the `instance-id` and applies tags.
5. Check the **Tags tab** in the EC2 console:
   - `LaunchDate` → current date (UTC).
   - `Owner` → `AutoTagLambda`.
6. In **CloudWatch Logs**, confirm:
   ```
   Instance i-0abcd1234ef567890 tagged with LaunchDate=2026-03-15 and Owner=AutoTagLambda
   ```

📸 *Screenshot:* ![alt text](screenshots/cloudwatch-logs.png)

---

## 🚀 Notes
- **Automation Benefit**: Ensures consistent tagging across all EC2 instances without manual intervention.
- **Custom Tags**: Extend the `Tags` list for more metadata (e.g., `Environment`, `Project`).
- **Security Best Practice**: Replace `AmazonEC2FullAccess` with a scoped IAM policy.

---
