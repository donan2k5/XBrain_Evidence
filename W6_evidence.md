# W6 Evidence Pack

---

## Section 1 — Cover

| Field | Details |
|---|---|
| **Group ID** | `GRP-XX` |
| **Thành viên** | Nguyễn Văn A · Trần Thị B · Lê Văn C |
| **Repository** | [https://github.com/your-org/your-repo](https://github.com/your-org/your-repo) |
| **W5 Evidence Pack** | [https://github.com/your-org/your-repo/blob/main/docs/W5_Evidence_Pack.md](https://github.com/your-org/your-repo/blob/main/docs/W5_Evidence_Pack.md) |

### W5 Feedback đã giải quyết *(tuỳ chọn)*

> *(Ví dụ: Đã bổ sung lại CloudWatch alarm threshold theo phản hồi của trainer — ngưỡng cũ 5xx > 10 req, đã sửa thành > 5 req với evaluation period 2 phút.)*

---

## Section 2 — MH-COST-V: Cost Visibility & Attribution

### 2.1 Screenshot — Tags bắt buộc trên ≥ 3 loại resource khác nhau

> **Yêu cầu:** Hiển thị đủ 4 key bắt buộc (`Project`, `Environment`, `Owner`, `CostCenter`) trên ít nhất ba loại resource khác nhau (ví dụ: EC2, RDS, S3, Lambda…).

**Resource 1 — EC2 Instance**

```
[Dán screenshot tại đây]
```

*Mô tả: EC2 instance `web-server-prod`, Region `ap-southeast-1`, hiển thị 4 tag bắt buộc.*

---

**Resource 2 — RDS Instance**

```
[Dán screenshot tại đây]
```

*Mô tả: RDS instance `db-prod`, hiển thị 4 tag bắt buộc.*

---

**Resource 3 — S3 Bucket**

```
[Dán screenshot tại đây]
```

*Mô tả: S3 bucket `your-bucket-name`, hiển thị 4 tag bắt buộc.*

---

### 2.2 Screenshot — Cost Allocation Tags Activated (Billing Console)

```
[Dán screenshot Billing Console → Cost Allocation Tags tại đây]
```

*Xác nhận: Cả 4 key (`Project`, `Environment`, `Owner`, `CostCenter`) ở trạng thái **Active** trong AWS Billing Console.*

---

### 2.3 Cost Explorer — Filter theo Tag + Baseline Cost Breakdown

**Cost Explorer filtered by tag:**

```
[Dán screenshot Cost Explorer đã filter theo tag của project tại đây]
```

**Baseline cost breakdown (by service):**

```
[Dán screenshot Cost by Service tại đây]
```

#### Quan sát — Top 3 Cost Driver

> *(Viết đoạn quan sát thực tế của nhóm sau khi xem breakdown. Ví dụ bên dưới — thay bằng số liệu thực của bạn.)*

Qua việc filter Cost Explorer theo tag `Project = ecommerce-platform`, nhóm xác định được ba dịch vụ chiếm phần lớn chi phí trong kỳ báo cáo. **Amazon EC2** là cost driver lớn nhất, chiếm khoảng 52 % tổng chi phí, chủ yếu do hai instance `t3.medium` chạy liên tục 24/7 cho môi trường production và staging. **Amazon RDS (MySQL)** đứng thứ hai với khoảng 28 %, phát sinh từ instance `db.t3.small` Multi-AZ được bật để đảm bảo high availability. **AWS Data Transfer** chiếm vị trí thứ ba (~12 %), bao gồm traffic egress từ ALB ra internet và giữa các Availability Zone — con số này cao hơn dự kiến và là ứng viên để tối ưu trong sprint tiếp theo.

---

### 2.4 Tagging Strategy Document

#### 1. Mục tiêu

Gắn tag nhất quán cho toàn bộ resource AWS để hỗ trợ cost attribution theo dự án, môi trường và chủ sở hữu; đồng thời phục vụ automated cost guard và báo cáo compliance.

#### 2. Tag Schema bắt buộc

| Key | Giá trị ví dụ | Bắt buộc? | Mô tả |
|---|---|---|---|
| `Project` | `ecommerce-platform` | ✅ | Tên dự án, không đổi |
| `Environment` | `prod` / `staging` / `dev` | ✅ | Môi trường triển khai |
| `Owner` | `team-backend` | ✅ | Team hoặc cá nhân chịu trách nhiệm |
| `CostCenter` | `CC-1042` | ✅ | Mã trung tâm chi phí kế toán |
| `ManagedBy` | `terraform` | ⬜ Khuyến nghị | Công cụ IaC quản lý resource |
| `AutoStop` | `true` / `false` | ⬜ Khuyến nghị | Cờ cho Lambda cost guard |

#### 3. Scope áp dụng

Tất cả resource trong tài khoản AWS của dự án: EC2, RDS, S3, Lambda, ALB, CloudWatch, EBS volumes, Elastic IP.

#### 4. Cơ chế thực thi

- **Terraform default tags:** Tag được inject tự động qua `default_tags` trong `provider "aws"` block.
- **AWS Config Rule `required-tags`:** Tự động phát hiện resource thiếu tag và sinh finding.
- **SCP (Service Control Policy):** Chặn tạo resource nếu thiếu tag `Project` và `Environment` (áp dụng ở cấp OU).

#### 5. Quy trình cập nhật

Bất kỳ thay đổi tag schema nào phải được tạo PR, review bởi ít nhất một thành viên khác, và cập nhật đồng thời vào tài liệu này + Terraform variables.

---

## Section 3 — MH-COST-A: Cost Control & Action

### 3.1 Automated Cost Guard — Lambda Code/Config

```python
# cost_guard/lambda_function.py
import boto3
import os

ec2 = boto3.client('ec2')
rds = boto3.client('rds')

STOP_TAG_KEY   = os.environ.get('STOP_TAG_KEY', 'AutoStop')
STOP_TAG_VALUE = os.environ.get('STOP_TAG_VALUE', 'true')

def lambda_handler(event, context):
    # Stop tagged EC2 instances
    ec2_response = ec2.describe_instances(
        Filters=[
            {'Name': f'tag:{STOP_TAG_KEY}', 'Values': [STOP_TAG_VALUE]},
            {'Name': 'instance-state-name',  'Values': ['running']}
        ]
    )
    instance_ids = [
        i['InstanceId']
        for r in ec2_response['Reservations']
        for i in r['Instances']
    ]
    if instance_ids:
        ec2.stop_instances(InstanceIds=instance_ids)
        print(f"Stopped EC2 instances: {instance_ids}")

    # Stop tagged RDS instances
    rds_response = rds.describe_db_instances()
    for db in rds_response['DBInstances']:
        tags_resp = rds.list_tags_for_resource(
            ResourceName=db['DBInstanceArn']
        )
        tags = {t['Key']: t['Value'] for t in tags_resp['TagList']}
        if tags.get(STOP_TAG_KEY) == STOP_TAG_VALUE \
                and db['DBInstanceStatus'] == 'available':
            rds.stop_db_instance(DBInstanceIdentifier=db['DBInstanceIdentifier'])
            print(f"Stopped RDS: {db['DBInstanceIdentifier']}")

    return {'statusCode': 200, 'body': 'Cost guard executed'}
```

**Screenshot — Lambda function configuration:**

```
[Dán screenshot Lambda Console (function name, runtime, timeout, env vars) tại đây]
```

---

### 3.2 Least-Privilege IAM Role

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EC2StopTagged",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:StopInstances"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ec2:ResourceTag/AutoStop": "true"
        }
      }
    },
    {
      "Sid": "RDSStop",
      "Effect": "Allow",
      "Action": [
        "rds:DescribeDBInstances",
        "rds:ListTagsForResource",
        "rds:StopDBInstance"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Logs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```

**Screenshot — IAM Role & attached policy:**

```
[Dán screenshot IAM Role summary tại đây]
```

---

### 3.3 EventBridge Daily Schedule

```
[Dán screenshot EventBridge rule (cron schedule, target Lambda, enabled status) tại đây]
```

*Cron expression ví dụ: `cron(0 18 * * ? *)` — chạy lúc 18:00 UTC mỗi ngày.*

---

### 3.4 Before/After — EC2/RDS bị stop bởi Lambda

**Before (instance running):**

```
[Dán screenshot EC2/RDS Console — instance ở trạng thái "running" / "available"]
```

**After (instance stopped):**

```
[Dán screenshot EC2/RDS Console — instance ở trạng thái "stopped"]
```

---

### 3.5 CloudTrail Event — `StopInstances` / `StopDBInstance`

```json
{
  "eventVersion": "1.08",
  "userIdentity": {
    "type": "AssumedRole",
    "principalId": "AROAXXXXXXXXXXXXXXXXX:cost-guard-lambda",
    "arn": "arn:aws:sts::123456789012:assumed-role/CostGuardLambdaRole/cost-guard-lambda"
  },
  "eventTime": "2025-05-20T18:00:05Z",
  "eventSource": "ec2.amazonaws.com",
  "eventName": "StopInstances",
  "awsRegion": "ap-southeast-1",
  "requestParameters": {
    "instancesSet": {
      "items": [{"instanceId": "i-0abc123def456"}]
    }
  },
  "responseElements": {
    "instancesSet": {
      "items": [{
        "instanceId": "i-0abc123def456",
        "currentState": {"code": 64, "name": "stopping"},
        "previousState": {"code": 16, "name": "running"}
      }]
    }
  }
}
```

```
[Dán screenshot CloudTrail event tại đây]
```

---

### 3.6 AWS Budgets — Daily $150 → SNS → Lambda Wiring

```
[Dán screenshot Budgets configuration (budget amount $150/day, SNS action) tại đây]
```

```
[Dán screenshot SNS topic subscription → Lambda tại đây]
```

---

### 3.7 Test SNS Publish — Demonstration

```bash
# Command dùng để test publish
aws sns publish \
  --topic-arn "arn:aws:sns:ap-southeast-1:123456789012:cost-alert-topic" \
  --message '{"AlarmName":"BudgetExceeded","NewStateValue":"ALARM"}' \
  --subject "Budget Threshold Test"
```

```
[Dán screenshot SNS publish result + Lambda invocation log tại đây]
```

---

### 3.8 Latency ADR — Cost Data Latency

**ADR-COST-001: Chấp nhận độ trễ Cost Explorer lên đến 24 giờ**

| Field | Detail |
|---|---|
| **Ngày** | 2025-05-20 |
| **Trạng thái** | Accepted |
| **Bối cảnh** | AWS Cost Explorer và Budgets có độ trễ dữ liệu lên đến 24 giờ. Cost guard Lambda không thể phản ứng ngay lập tức với chi phí phát sinh trong ngày. |
| **Quyết định** | Chạy Lambda theo daily schedule (18:00 UTC) dựa trên tag `AutoStop=true`, không phụ thuộc real-time cost data. |
| **Hệ quả** | Một resource không được tag đúng có thể chạy thêm tối đa 24 giờ trước khi bị phát hiện. Trade-off được chấp nhận vì cost impact ở quy mô dự án là thấp (< $5/ngày chênh lệch). |
| **Giải pháp thay thế đã xem xét** | AWS Cost Anomaly Detection (độ trễ tương tự, thêm overhead cấu hình). |

---

## Section 4 — MH-OBS: CloudWatch Observability

### 4.1 Dashboard — Ba loại widget

```
[Dán screenshot CloudWatch Dashboard tại đây — đảm bảo cả 3 widget type hiển thị rõ:
  1. Metric widget (Line/Number) — gắn nhãn tường minh, ví dụ "API 5xx Error Rate"
  2. Alarm widget — hiển thị trạng thái OK/ALARM
  3. Log widget (Log Insights query kết quả)
]
```

*Widget titles được dùng:*
- **Custom metric widget:** `AppOrdersCreated` — số đơn hàng tạo mới mỗi phút (namespace `EcommApp/Orders`)
- **Alarm widget:** `HighErrorRate-Alarm`
- **Log Insights widget:** `RecentAPIErrors`

---

### 4.2 Code — `PutMetricData` từ ứng dụng

```python
# app/metrics.py
import boto3
import time

cloudwatch = boto3.client('cloudwatch', region_name='ap-southeast-1')

def emit_order_created_metric(order_id: str):
    cloudwatch.put_metric_data(
        Namespace='EcommApp/Orders',
        MetricData=[
            {
                'MetricName': 'OrdersCreated',
                'Dimensions': [
                    {'Name': 'Environment', 'Value': 'prod'},
                    {'Name': 'Service',     'Value': 'order-service'},
                ],
                'Value':     1,
                'Unit':      'Count',
                'Timestamp': time.time(),
            }
        ]
    )
    print(f"[Metrics] OrdersCreated emitted for order {order_id}")
```

---

### 4.3 Alarm Configuration

```
[Dán screenshot CloudWatch Alarm configuration tại đây]
```

| Field | Value |
|---|---|
| **Metric name** | `5XXError` (namespace `AWS/ApiGateway`) |
| **Threshold** | `> 5` trong 1 evaluation period |
| **Evaluation period** | 1 phút (60 giây) |
| **Datapoints to alarm** | 1 of 1 |
| **Action destination** | SNS topic `ops-alerts` → email nhóm |
| **Alarm state** | `OK` *(hoặc `ALARM` nếu đang có lỗi)* |

---

### 4.4 Log Insights Query

**Query text:**

```sql
fields @timestamp, @message, status, path
| filter status >= 500
| sort @timestamp desc
| limit 20
```

**Log group:** `/aws/apigateway/your-api-name`  
**Saved query name:** `RecentServerErrors`

```
[Dán screenshot Log Insights — query text + log group + ≥ 5 result row tại đây]
```

```
[Dán screenshot Saved Queries list với "RecentServerErrors" visible tại đây]
```

---

## Section 5 — MH-SEC: Self-Healing Security Guard

### 5.1 Lambda Code/Config

```python
# security_guard/lambda_function.py
import boto3
import json

ec2 = boto3.client('ec2')
s3  = boto3.client('s3')

def lambda_handler(event, context):
    detail = event.get('detail', {})
    event_name = detail.get('eventName', '')

    if event_name == 'AuthorizeSecurityGroupIngress':
        _remediate_sg(detail)
    elif event_name == 'PutBucketAcl':
        _remediate_s3_acl(detail)

    return {'statusCode': 200}

def _remediate_sg(detail):
    sg_id = detail['requestParameters']['groupId']
    ip_permissions = detail['requestParameters'].get('ipPermissions', {})
    items = ip_permissions.get('items', [])
    for rule in items:
        for ip_range in rule.get('ipRanges', {}).get('items', []):
            if ip_range.get('cidrIp') == '0.0.0.0/0':
                print(f"[SEC] Revoking 0.0.0.0/0 rule on {sg_id}")
                ec2.revoke_security_group_ingress(
                    GroupId=sg_id,
                    IpPermissions=[{
                        'IpProtocol': rule['ipProtocol'],
                        'FromPort':   rule.get('fromPort', 0),
                        'ToPort':     rule.get('toPort', 65535),
                        'IpRanges':   [{'CidrIp': '0.0.0.0/0'}]
                    }]
                )

def _remediate_s3_acl(detail):
    bucket = detail['requestParameters']['bucketName']
    print(f"[SEC] Enabling S3 Block Public Access on {bucket}")
    s3.put_public_access_block(
        Bucket=bucket,
        PublicAccessBlockConfiguration={
            'BlockPublicAcls':       True,
            'IgnorePublicAcls':      True,
            'BlockPublicPolicy':     True,
            'RestrictPublicBuckets': True,
        }
    )
```

**Screenshot — Lambda function:**

```
[Dán screenshot Lambda Console tại đây]
```

---

### 5.2 Least-Privilege IAM Role (Security Guard)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SGRemediation",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeSecurityGroups",
        "ec2:RevokeSecurityGroupIngress"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3Remediation",
      "Effect": "Allow",
      "Action": [
        "s3:PutPublicAccessBlock",
        "s3:GetPublicAccessBlock"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudTrailRead",
      "Effect": "Allow",
      "Action": ["cloudtrail:LookupEvents"],
      "Resource": "*"
    },
    {
      "Sid": "Logs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```

**Screenshot — IAM Role:**

```
[Dán screenshot IAM Role tại đây]
```

---

### 5.3 Trigger — EventBridge Rule

```
[Dán screenshot EventBridge rule tại đây]
```

*Rule pattern ví dụ (trigger trên CloudTrail API call):*

```json
{
  "source": ["aws.ec2"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["ec2.amazonaws.com"],
    "eventName": ["AuthorizeSecurityGroupIngress"]
  }
}
```

---

### 5.4 Demo — Detect → Fix Loop

**Before (insecure — SG với rule 0.0.0.0/0 port 22):**

```
[Dán screenshot Security Group inbound rules — CÓ rule 0.0.0.0/0:22 tại đây]
```

**After (remediated — rule đã bị revoke):**

```
[Dán screenshot Security Group inbound rules — KHÔNG còn rule 0.0.0.0/0 tại đây]
```

**CloudTrail event — `RevokeSecurityGroupIngress`:**

```json
{
  "eventName": "RevokeSecurityGroupIngress",
  "userIdentity": {
    "type": "AssumedRole",
    "arn": "arn:aws:sts::123456789012:assumed-role/SecurityGuardLambdaRole/security-guard"
  },
  "eventTime": "2025-05-20T10:15:32Z",
  "requestParameters": {
    "groupId": "sg-0abc123def",
    "ipPermissions": {
      "items": [{
        "ipProtocol": "tcp",
        "fromPort": 22,
        "toPort": 22,
        "ipRanges": {"items": [{"cidrIp": "0.0.0.0/0"}]}
      }]
    }
  }
}
```

```
[Dán screenshot CloudTrail event tại đây]
```

---

### 5.5 Supporting Preventive Control

> *(Chọn một trong ba option bên dưới và điền evidence tương ứng. Xoá hai option còn lại.)*

---

#### Option A — KMS CMK

```
[Dán screenshot KMS CMK key policy + evidence kms:GenerateDataKey / kms:Decrypt từ data store tại đây]
```

---

#### Option B — S3 Block Public Access (Account-level) + Deny Policy

```
[Dán screenshot S3 Block Public Access — account level bật tại đây]
```

```
[Dán screenshot test call bị deny (AccessDenied) khi cố gắng set public ACL tại đây]
```

---

#### Option C — IAM Access Analyzer

```
[Dán screenshot IAM Access Analyzer finding (external access) tại đây]
```

*Triage decision:* *(Ghi rõ finding là legitimate hay cần fix, và hành động đã thực hiện.)*

---

### 5.6 Security Threat Statement

**Guard fix misconfiguration gì?**

Lambda Security Guard phát hiện và revoke bất kỳ inbound rule Security Group nào mở port SSH (22) hoặc tất cả port (`-1`) với CIDR `0.0.0.0/0` (toàn bộ internet). Misconfiguration này thường phát sinh do lỗi cấu hình thủ công hoặc infrastructure-as-code không đúng.

**Blast radius nếu không remediate?**

Nếu rule `0.0.0.0/0:22` tồn tại và không bị revoke: toàn bộ máy chủ EC2 trong Security Group đó bị lộ SSH ra internet công khai. Attacker có thể thực hiện brute-force, khai thác CVE trên SSH daemon, hoặc nếu chiếm được quyền truy cập sẽ lateral move vào toàn bộ VPC, truy cập database, exfiltrate dữ liệu khách hàng, và tiêu tốn tài nguyên cho cryptomining. Thời gian phát hiện thủ công (nếu không có guard) ước tính từ vài giờ đến vài ngày.

---

### 5.7 Security-Cost Trade-off Statement

> Việc chạy Lambda Security Guard liên tục 24/7 theo trigger EventBridge phát sinh chi phí Lambda invocation và CloudWatch Logs ước tính **< $1/tháng** ở quy mô dự án — hoàn toàn xứng đáng so với chi phí incident response, data breach notification, và thiệt hại reputational nếu một EC2 instance bị compromise do misconfigured Security Group. KMS CMK bổ sung thêm **$1/key/tháng** nhưng đảm bảo data-at-rest encryption có audit trail đầy đủ, đáp ứng yêu cầu compliance. Tổng chi phí preventive control **~$2/tháng** là mức trade-off hợp lý và được nhóm chấp nhận.

---

## Section 6 — Project Recap

### Ứng dụng là gì?

> *(Mô tả ngắn gọn ứng dụng — ví dụ bên dưới, thay bằng nội dung thực của nhóm.)*

Nhóm xây dựng nền tảng **e-commerce đơn giản** phục vụ thị trường bán lẻ B2C, bao gồm các tính năng: duyệt sản phẩm, quản lý giỏ hàng, đặt hàng và thanh toán trực tuyến. Business domain là **retail commerce**, hướng tới SMB muốn digital hoá hoạt động bán hàng mà không cần đầu tư hạ tầng lớn.

### Quyết định kiến trúc và thiết kế chính (W1–W5)

- **W1–W2:** Kiến trúc three-tier (ALB → EC2 Auto Scaling Group → RDS MySQL Multi-AZ). Chọn region `ap-southeast-1` (Singapore) để tối ưu latency cho người dùng Đông Nam Á.
- **W3:** Tách stateless application layer ra khỏi database layer; sử dụng S3 + CloudFront cho static assets. Implement CI/CD pipeline với GitHub Actions → CodeDeploy.
- **W4:** Bổ sung VPC với public/private subnet, NAT Gateway, Security Group least-privilege. RDS đặt trong private subnet, chỉ EC2 trong App SG được kết nối.
- **W5:** Triển khai API Gateway + Lambda cho một số service phụ (email notification, order status webhook). Bổ sung CloudWatch basic monitoring và X-Ray tracing.
- **W6 (hiện tại):** Thêm lớp vận hành: cost tagging, automated cost guard, CloudWatch Observability dashboard, và self-healing security Lambda.

### W5 Feedback đã giải quyết *(tuỳ chọn)*

> *(Ví dụ: Trainer feedback W5 — CloudWatch alarm thiếu action. Đã bổ sung SNS action → email alert trong W6.)*

---

## Bonus Section *(tuỳ chọn)*

### Bonus Item: *(Tên bonus item)*

**Before:**

```
[Dán screenshot trước khi thực hiện bonus tại đây]
```

**After:**

```
[Dán screenshot sau khi thực hiện bonus tại đây]
```

**Đo lường:**

| Metric | Before | After |
|---|---|---|
| *(ví dụ: Response time)* | `XXX ms` | `YYY ms` |

**Reflection:**

> *(2–3 câu reflection về bonus item đã thực hiện — những gì học được, kết quả đạt được, và điều gì có thể cải thiện thêm.)*

---

*— Kết thúc W6 Evidence Pack —*