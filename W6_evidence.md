# W6 Evidence Pack

---

## Section 1 — Cover

### Thành viên

| No. | Họ tên              | Nhiệm vụ W6                                                      |
| --- | ------------------- | ---------------------------------------------------------------- |
| 1   | Nguyễn Thành Đạt    | MH-COST-A   |
| 2   | Hoàng Minh Hải      | MH-COST-A |
| 3   | Đinh Văn Ty         | MH-COST-V                 |
| 5   | Từ Phúc Nguyên | MH-COST-V       |
| 6   | Phan Lê Thanh Hoàng| MH-SEC          |
| 7   | Nguyễn Văn Toàn     | MH-SEC             |
| 8   | Nguyễn Minh Thanh   | MH-OBS              |
| 9   | Ngô Thanh Kiên      | MH-OBS 
| 10  | Lê Trần Ánh Nhung   | MH-OBS

### W5 Feedback đã giải quyết *(tuỳ chọn)*

> *(Ví dụ: Đã bổ sung lại CloudWatch alarm threshold theo phản hồi của trainer — ngưỡng cũ 5xx > 10 req, đã sửa thành > 5 req với evaluation period 2 phút.)*

---

## Section 2 — MH-COST-V: Cost Visibility & Attribution

### 2.1 Screenshot — Tags bắt buộc trên ≥ 3 loại resource khác nhau

> **Yêu cầu:** Hiển thị đủ 4 key bắt buộc (`g3:owner`, `g3:environment`, `g3:cost-center	`, `g3:application`) trên ít nhất ba loại resource khác nhau (ví dụ: EC2, RDS, S3, Lambda…).

**Resource 1 — EC2 Instance**

![image](https://hackmd.io/_uploads/SJNogNTyMe.png)



*Mô tả: EC2 instance có đầy đủ các tags theo như tagging document*

---

**Resource 2 — RDS Instance**
![image](https://hackmd.io/_uploads/Hk1TeVp1fl.png)



*Mô tả: RDS instance có đầy đủ các tags theo như tagging document*

---

**Resource 3 — S3 Bucket**

![image](https://hackmd.io/_uploads/SktulEpyGe.png)


*Mô tả: S3 bucket có đầy đủ các tags theo như tagging document*

---

**Resource 4 - Lambda**
![image](https://hackmd.io/_uploads/ryR7Z46yGg.png)

*Mô tả: Lambda có đầy đủ các tags theo như tagging document*


### 2.2 Cost Explorer



### 2.3 Cost Budget:

![image](https://hackmd.io/_uploads/Skn7BUaJfe.png)
Bud
![image](https://hackmd.io/_uploads/rJCHIUTJMg.png)

*Mô tả: Vì tổng budget của tuần này là 150$ 
, nên nhóm đã tạo 1 budget 50$ per day từ 20/5->22/5. Với 2 mức cảnh báo 25$ và 50$, khi chạm ngưỡng này sẽ gửi email cho tydv và nguyentp (2 người chịu trách nhiệm về chi phí cho project)*
### 2.4 Tagging Strategy Document

#### 1. Mục tiêu

Gắn tag nhất quán cho toàn bộ resource AWS để hỗ trợ cost attribution theo dự án, môi trường và chủ sở hữu; đồng thời phục vụ automated cost guard và báo cáo compliance.

#### 2. Tag Schema bắt buộc
Note: tất cả giá trị của tag đều viết bằng chữ thường (ngoại trừ timestamp).
| Key (Chuẩn Namespace)  | Value mẫu | Required? | Mô tả | Quy tắc áp dụng |
| :--- | :--- | :---: | :--- | :--- |
| Owner  | tydv, namn, ... | ✅Bắt buộc| Người sở hữu và chịu trách nhiệm tắt/bảo trì tài nguyên. | Tên + chữ cái đầu của họ và tên lót (Ví dụ: Đinh Văn Ty $\rightarrow$ tydv). |
| Environment  | prod, qc, dev | ✅Bắt buộc| Môi trường triển khai hệ thống. | Tuần này phục vụ redeploy chọn giá trị dev. |
| CostCenter  | g3 | ✅Bắt buộc| Mã phân tách tài chính của nhóm. | Cố định là chữ g3 viết thường. |
| Application  | ecommerce | ✅Bắt buộc| Resource phục vụ cho application nào | Mặc định là ecommerce vì nhóm chỉ đang triển khai 1 application |
| Keep | true, false | ✅Bắt buộc| Cờ hiệu cho Lambda cost guard quét dọn. | true: Giữ lại; false hoặc thiếu tag: Tự động tắt máy chủ vào 23h00. |
| CostGuardStatus  | pending-stop | ⚠️Không điền lúc tạo resource| Cờ hiệu cho Lambda biết resource đã được quét chưa. | pending-stop: Resource đã được quét, còn nếu không có tag thì chưa được quét.|
| FirstDetectedAt | 2026-05-22T08:15:30+00:00 | ⚠️Không điền lúc tạo resource| Timestamp mà Lambda đã quét qua resource. |Theo chuẩn ISO 8601 UTC timestamp  |
|Name|g3-dev-frontend-ec2,g3-dev-database-rds|⚠️Khuyến nghị|Tên hiển thị trực quan của tài nguyên trên AWS Console.|Đặt theo cú pháp ghép chuỗi (viết thường 100%, cách nhau bằng dấu gạch ngang):[nhóm]-[môi trường]-[tầng app]-[loại tài nguyên]|

### 2.5 Cost governance 

#### 3. Scope áp dụng

Tất cả resource trong tài khoản AWS của dự án: EC2, RDS, S3, Lambda, ALB, CloudWatch, EBS volumes, Elastic IP.

---

## Section 3 — MH-COST-A: Cost Control & Action

### 3.1 Automated Cost Guard — Lambda Code/Config

```python
# cost_guard/lambda_function.py
import boto3
import logging
import json
import os
from datetime import datetime, timezone

logger = logging.getLogger()
logger.setLevel(logging.INFO)

# ---------------------------------------------------------
# CẤU HÌNH TAGGING STANDARD
# ---------------------------------------------------------
REQUIRED_KEYS = ['Owner', 'Environment', 'CostCenter', 'Application']
KEEP_TAG_KEY = os.environ.get('KEEP_TAG_KEY', 'Keep')
KEEP_TAG_VALUE = os.environ.get('KEEP_TAG_VALUE', 'true')
WARNING_SNS_ARN = os.environ.get('WARNING_SNS_ARN', '')
GRACE_PERIOD_HOURS = 2

def lambda_handler(event, context):
    logger.info(f"Trigger Event: {json.dumps(event)}")
    
    is_budget_alert = False
    if 'Records' in event and 'Sns' in event['Records'][0]:
        message_id = event['Records'][0]['Sns'].get('MessageId', '')
        if 'simulate' not in message_id:
            is_budget_alert = True
            logger.warning("AWS BUDGETS ALERT: Kích hoạt chế độ tắt khẩn cấp.")

    ec2_client = boto3.client('ec2')
    rds_client = boto3.client('rds')
    sns_client = boto3.client('sns')
    
    process_ec2_state_machine(ec2_client, sns_client, is_budget_alert)
    process_rds_state_machine(rds_client, sns_client, is_budget_alert)

    return {'statusCode': 200, 'body': 'Executed successfully'}

def analyze_tags(tags_dict):
    """
    Phân tách 2 trạng thái: Có được giữ máy không (keep_valid) và Thiếu tag gì (missing_info)
    """
    # 1. Kiểm tra quyền sinh sát (g3:keep)
    current_keep = tags_dict.get(KEEP_TAG_KEY, '')
    keep_valid = (current_keep.lower() == KEEP_TAG_VALUE.lower())
    
    # 2. Kiểm tra các tag thông tin
    missing_info = [key for key in REQUIRED_KEYS if not tags_dict.get(key, '').strip()]
    
    return keep_valid, missing_info

def send_email(sns, severity, res_type, res_id, timestamp_str, missing_info):
    """Hàm phụ trợ để gửi email tùy theo mức độ nghiêm trọng"""
    if not WARNING_SNS_ARN: return
    
    missing_str = ", ".join(missing_info) if missing_info else "Không có"
    
    if severity == "CRITICAL":
        subject = f"[KHẨN CẤP] AWS FinOps: {res_type} {res_id} SẮP BỊ TẮT"
        message = (f"Tài nguyên {res_type} ({res_id}) ĐANG THIẾU thẻ giữ máy '{KEEP_TAG_KEY}={KEEP_TAG_VALUE}'.\n\n"
                   f"Hệ thống sẽ TỰ ĐỘNG TẮT tài nguyên này sau {GRACE_PERIOD_HOURS} giờ nữa.\n"
                   f"Thời gian phát hiện: {timestamp_str} UTC.\n\n"
                   f"Các Tag quản lý cũng đang bị thiếu: {missing_str}\n\n"
                   f"Vui lòng bổ sung tag '{KEEP_TAG_KEY}={KEEP_TAG_VALUE}' ngay lập tức để bảo vệ tài nguyên!")
    else: # WARNING
        subject = f"[NHẮC NHỞ] AWS FinOps: {res_type} {res_id} thiếu Tag quản lý"
        message = (f"Tài nguyên {res_type} ({res_id}) ĐÃ CÓ thẻ '{KEEP_TAG_KEY}={KEEP_TAG_VALUE}' (An toàn, KHÔNG BỊ TẮT).\n\n"
                   f"TUY NHIÊN, bạn đang thiếu các Tag thông tin bắt buộc theo chuẩn của dự án:\n"
                   f"- {missing_str}\n\n"
                   f"Vui lòng bổ sung các tag này để hoàn thiện thông tin quản lý chi phí.\n"
                   f"Cảm ơn sự hợp tác của bạn.")
                   
    sns.publish(TopicArn=WARNING_SNS_ARN, Subject=subject, Message=message)

def process_ec2_state_machine(ec2, sns, is_budget_alert):
    now = datetime.now(timezone.utc)
    response = ec2.describe_instances(Filters=[{'Name': 'instance-state-name', 'Values': ['running']}])
    
    for reservation in response['Reservations']:
        for instance in reservation['Instances']:
            instance_id = instance['InstanceId']
            tags = {tag['Key']: tag['Value'] for tag in instance.get('Tags', [])}
            
            keep_valid, missing_info = analyze_tags(tags)
            current_status = tags.get('CostGuardStatus')
            
            # --- 1. BUDGET ALERT (Thủng ngân sách) ---
            if is_budget_alert:
                if not keep_valid:
                    ec2.stop_instances(InstanceIds=[instance_id])
                    logger.info(f"EC2 {instance_id}: STOPPED (Budget Alert).")
                continue

            # --- 2. LỖI CHÍ MẠNG (Thiếu keep=true) -> Cảnh báo & Tắt ---
            if not keep_valid:
                if current_status == 'PendingStop':
                    # Kiểm tra đếm ngược
                    detected_str = tags.get('FirstDetectedAt')
                    if detected_str:
                        try:
                            detected_time = datetime.fromisoformat(detected_str)
                            hours_passed = (now - detected_time).total_seconds() / 3600
                            if hours_passed >= GRACE_PERIOD_HOURS:
                                ec2.stop_instances(InstanceIds=[instance_id])
                                logger.info(f"EC2 {instance_id}: Hết ân hạn. TIẾN HÀNH TẮT.")
                        except ValueError: pass
                else:
                    # Vi phạm lần đầu -> Gắn PendingStop & Gửi Email KHẨN CẤP
                    timestamp_str = now.isoformat()
                    ec2.create_tags(Resources=[instance_id], Tags=[
                        {'Key': 'CostGuardStatus', 'Value': 'PendingStop'},
                        {'Key': 'FirstDetectedAt', 'Value': timestamp_str}
                    ])
                    logger.info(f"EC2 {instance_id}: Đánh dấu PendingStop.")
                    send_email(sns, "CRITICAL", "EC2", instance_id, timestamp_str, missing_info)

            # --- 3. AN TOÀN (Đã có keep=true) -> KHÔNG TẮT ---
            else: 
                if missing_info:
                    # LỖI CẢNH CÁO: Đủ keep=true nhưng thiếu tag thông tin -> Gửi nhắc nhở LIÊN TỤC
                    # Xóa tag phạt cũ (nếu có) để dọn dẹp hệ thống
                    if current_status == 'PendingStop':
                        ec2.delete_tags(Resources=[instance_id], Tags=[{'Key': 'CostGuardStatus'}, {'Key': 'FirstDetectedAt'}])
                    
                    logger.info(f"EC2 {instance_id}: Thiếu tag thông tin (Gửi nhắc nhở liên tục).")
                    send_email(sns, "WARNING", "EC2", instance_id, None, missing_info)
                else:
                    # HỢP LỆ 100%: Đầy đủ mọi tag
                    if current_status:
                        ec2.delete_tags(Resources=[instance_id], Tags=[{'Key': 'CostGuardStatus'}, {'Key': 'FirstDetectedAt'}])
                        logger.info(f"EC2 {instance_id}: Tuân thủ 100%. Đã gỡ bỏ mọi án phạt/nhắc nhở.")


def process_rds_state_machine(rds, sns, is_budget_alert):
    now = datetime.now(timezone.utc)
    response = rds.describe_db_instances()
    
    for db in response['DBInstances']:
        if db['DBInstanceStatus'] != 'available': continue
        
        db_id = db['DBInstanceIdentifier']
        db_arn = db['DBInstanceArn']
        tags = {tag['Key']: tag['Value'] for tag in db.get('TagList', [])}
        
        keep_valid, missing_info = analyze_tags(tags)
        current_status = tags.get('CostGuardStatus')
        
        if is_budget_alert:
            if not keep_valid:
                rds.stop_db_instance(DBInstanceIdentifier=db_id)
                logger.info(f"RDS {db_id}: STOPPED (Budget Alert).")
            continue

        if not keep_valid:
            if current_status == 'PendingStop':
                detected_str = tags.get('FirstDetectedAt')
                if detected_str:
                    try:
                        detected_time = datetime.fromisoformat(detected_str)
                        hours_passed = (now - detected_time).total_seconds() / 3600
                        if hours_passed >= GRACE_PERIOD_HOURS:
                            rds.stop_db_instance(DBInstanceIdentifier=db_id)
                            logger.info(f"RDS {db_id}: Hết ân hạn. TIẾN HÀNH TẮT.")
                    except ValueError: pass
            else:
                timestamp_str = now.isoformat()
                rds.add_tags_to_resource(ResourceName=db_arn, Tags=[
                    {'Key': 'CostGuardStatus', 'Value': 'PendingStop'},
                    {'Key': 'FirstDetectedAt', 'Value': timestamp_str}
                ])
                logger.info(f"RDS {db_id}: Đánh dấu PendingStop.")
                send_email(sns, "CRITICAL", "RDS", db_id, timestamp_str, missing_info)
        else: 
            if missing_info:
                if current_status == 'PendingStop':
                    rds.remove_tags_from_resource(ResourceName=db_arn, TagKeys=['CostGuardStatus', 'FirstDetectedAt'])
                logger.info(f"RDS {db_id}: Thiếu tag thông tin (Gửi nhắc nhở liên tục).")
                send_email(sns, "WARNING", "RDS", db_id, None, missing_info)
            else:
                if current_status:
                    rds.remove_tags_from_resource(ResourceName=db_arn, TagKeys=['CostGuardStatus', 'FirstDetectedAt'])
                    logger.info(f"RDS {db_id}: Tuân thủ 100%. Gỡ bỏ phạt.")
```
Logic hiện tại hoạt động như một State Machine để kiểm soát chi phí và enforce tagging standard cho EC2/RDS. Lambda sẽ chạy định kỳ (ví dụ mỗi giờ) hoặc được trigger bởi AWS Budget Alert.
1. Xác định context chạy
* Nếu chạy từ EventBridge → kiểm tra định kỳ bình thường.
* Nếu chạy từ SNS Budget Alert → bật chế độ khẩn cấp (is_budget_alert=True).
2. Scan tài nguyên
* Lấy toàn bộ:
    * EC2 đang running
    * RDS đang available
* Sau đó kiểm tra tag của từng resource.
3. Phân tích tag
Hệ thống kiểm tra 2 thứ:
* Resource có Keep=true không → quyết định có được “giữ mạng” hay không.
* Có thiếu các tag quản lý bắt buộc (Name, owner, environment, cost-center, ...) không.
4. Ra quyết định

**Trường hợp 1 — Budget Alert**

Nếu đang vượt ngân sách:

Resource nào không có Keep=true
→ bị stop ngay lập tức, không grace period.

**Trường hợp 2 — Thiếu Keep=true**

Đây là lỗi nghiêm trọng.

* Nếu mới phát hiện:
    * Gắn tag:
        * CostGuardStatus=PendingStop
        * FirstDetectedAt=<timestamp>
    * Gửi email cảnh báo khẩn cấp.
    * Bắt đầu countdown 2 giờ.
* Nếu đã bị đánh dấu trước đó:
    * Kiểm tra đã quá 2 giờ chưa.
    * Nếu quá → tự động stop resource.
    
**Trường hợp 3 — Có Keep=true**

Resource sẽ không bị stop.

Tuy nhiên:

* Nếu thiếu các tag quản lý khác:
    * Gửi email nhắc nhở liên tục mỗi lần Lambda chạy.
    * Đồng thời xóa các tag “PendingStop” cũ nếu có.
* Nếu đầy đủ toàn bộ tag:
    * Xóa toàn bộ tag trạng thái/phạt.
    * Xem như compliant hoàn toàn.

Tóm lại, hệ thống này dùng tag để: quyết định tài nguyên có được giữ lại hay không,
tự động cleanup tài nguyên gây tốn chi phí,
và ép Dev tuân thủ chuẩn tagging của dự án thông qua warning + auto-stop.

**Screenshot — Lambda function configuration:**
![image](https://hackmd.io/_uploads/SJrkcb61Mx.png)
![image](https://hackmd.io/_uploads/SkJCKba1Me.png)


---

### 3.2 Least-Privilege IAM Role

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowDescribeResources",
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "rds:DescribeDBInstances"
            ],
            "Resource": "*"
        },
        {
            "Sid": "AllowStopAndTagEC2",
            "Effect": "Allow",
            "Action": [
                "ec2:StopInstances",
                "ec2:CreateTags",
                "ec2:DeleteTags"
            ],
            "Resource": "arn:aws:ec2:*:*:instance/*"
        },
        {
            "Sid": "AllowStopAndTagRDS",
            "Effect": "Allow",
            "Action": [
                "rds:StopDBInstance",
                "rds:AddTagsToResource",
                "rds:RemoveTagsFromResource"
            ],
            "Resource": "arn:aws:rds:*:*:db:*"
        },
        {
            "Sid": "AllowPublishWarning",
            "Effect": "Allow",
            "Action": "sns:Publish",
            "Resource": "arn:aws:sns:*:*:Dev-Warning-Topic"
        },
        {
            "Sid": "AllowCloudWatchLogs",
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

![image](https://hackmd.io/_uploads/Sy3u5Wakzg.png)


---

### 3.3 EventBridge Daily Schedule
    
Tạo Schedule chạy 1 lần/giờ

![image](https://hackmd.io/_uploads/rJn09ZTkMe.png)
![image](https://hackmd.io/_uploads/HyMloZTyMg.png)


---

### 3.4 Before/After — EC2/RDS bị stop bởi Lambda

**Before (instance running):**

**EC2 có đủ tag**
![image](https://hackmd.io/_uploads/SkkFnbpkGg.png)

**EC2 không có đủ tag**
![image](https://hackmd.io/_uploads/SkYR3bakfx.png)
    
**EC2 chỉ có tag keep = true**
![image](https://hackmd.io/_uploads/ryx-6bT1fx.png)


**After (instance stopped):**

**EC2 có đủ tag** -> Không có gì xảy ra vẫn hoạt động bình thường
![image](https://hackmd.io/_uploads/BJEsp-6kMx.png)

**EC2 không có đủ tag** -> chưa bị stop nhưng bị gắn tag CostGuardStatus và FirstDetectedAt và gửi mail thông báo yêu cầu phải sửa nếu muốn giữ instance này, nếu sau 2 giờ mà không sửa thì instance này sẽ bị stop
    
![image](https://hackmd.io/_uploads/BksKPEpkfl.png)
![image](https://hackmd.io/_uploads/Byt20ZpJGe.png)
    
Nếu sau 2 giờ mà chưa sửa thì instance sẽ bị stop:
    
![image](https://hackmd.io/_uploads/r1Jrkza1Ml.png)

**EC2 chỉ có tag keep = true** -> vẫn chạy bình thường nhưng sẽ nhận được mail yêu cầu bổ sung tag
    
![image](https://hackmd.io/_uploads/Sk4cJG61Gg.png)


---

### 3.5 CloudTrail Event — `StopInstances` / `StopDBInstance`

```json
{
    "eventVersion": "1.11",
    "userIdentity": {
        "type": "AssumedRole",
        "principalId": "AROATQWUQYEKCGAFPUATL:MH-COST-A-AutoStop",
        "arn": "arn:aws:sts::242037604628:assumed-role/CostControl-Lambda-Role/MH-COST-A-AutoStop",
        "accountId": "242037604628",
        "accessKeyId": "ASIATQWUQYEKGVBAXMQC",
        "sessionContext": {
            "sessionIssuer": {
                "type": "Role",
                "principalId": "AROATQWUQYEKCGAFPUATL",
                "arn": "arn:aws:iam::242037604628:role/CostControl-Lambda-Role",
                "accountId": "242037604628",
                "userName": "CostControl-Lambda-Role"
            },
            "attributes": {
                "creationDate": "2026-05-21T23:02:19Z",
                "mfaAuthenticated": "false"
            }
        },
        "inScopeOf": {
            "issuerType": "AWS::Lambda::Function",
            "credentialsIssuedTo": "arn:aws:lambda:us-east-1:242037604628:function:MH-COST-A-AutoStop"
        }
    },
    "eventTime": "2026-05-21T23:02:23Z",
    "eventSource": "ec2.amazonaws.com",
    "eventName": "StopInstances",
    "awsRegion": "us-east-1",
    "sourceIPAddress": "3.88.176.191",
    "userAgent": "Boto3/1.40.4 md/Botocore#1.40.4 ua/2.1 os/linux#5.10.252-285.992.amzn2.aarch64 md/arch#aarch64 lang/python#3.14.3 md/pyimpl#CPython exec-env/AWS_Lambda_python3.14 m/Z,D,b cfg/retry-mode#legacy Botocore/1.40.4",
    "requestParameters": {
        "instancesSet": {
            "items": [
                {
                    "instanceId": "i-063f3d1125b9264d7"
                }
            ]
        },
        "force": false,
        "skipOsShutdown": false
    },
    "responseElements": {
        "requestId": "34e44d98-9861-407b-941e-e3db3f1a9a4b",
        "instancesSet": {
            "items": [
                {
                    "instanceId": "i-063f3d1125b9264d7",
                    "currentState": {
                        "code": 64,
                        "name": "stopping"
                    },
                    "previousState": {
                        "code": 16,
                        "name": "running"
                    }
                }
            ]
        }
    },
    "requestID": "34e44d98-9861-407b-941e-e3db3f1a9a4b",
    "eventID": "20780ef3-c645-42d7-b4f8-12e569dc6f9a",
    "readOnly": false,
    "resources": [
        {
            "accountId": "242037604628",
            "type": "AWS::EC2::Instance",
            "ARN": "arn:aws:ec2:us-east-1:242037604628:instance/i-063f3d1125b9264d7"
        }
    ],
    "eventType": "AwsApiCall",
    "managementEvent": true,
    "recipientAccountId": "242037604628",
    "eventCategory": "Management",
    "tlsDetails": {
        "tlsVersion": "TLSv1.3",
        "cipherSuite": "TLS_AES_128_GCM_SHA256",
        "clientProvidedHostHeader": "ec2.us-east-1.amazonaws.com"
    }
}
```

![image](https://hackmd.io/_uploads/HJE7lG6yzg.png)


---

### 3.6 AWS Budgets — Daily $150 → SNS → Lambda Wiring

![image](https://hackmd.io/_uploads/rJQXbG61fx.png)
![image](https://hackmd.io/_uploads/rJcEbMT1fx.png)


**SNS Configuration**

![image](https://hackmd.io/_uploads/rJhcZz6yGe.png)

![image](https://hackmd.io/_uploads/rkalGzTJGx.png)



---

### 3.7 Test SNS Publish — Demonstration

```bash
# Command dùng để test publish
aws sns publish \
  --topic-arn "arn:aws:sns:ap-southeast-1:123456789012:cost-alert-topic" \
  --message '{"AlarmName":"BudgetExceeded","NewStateValue":"ALARM"}' \
  --subject "AWS Budgets: Alert for HardCap-Sandbox-150"
```

![image](https://hackmd.io/_uploads/rkUSrGakMg.png)



---

### 3.8 Latency ADR — Cost Data Latency

**ADR-COST-001: Chấp nhận độ trễ Cost Explorer lên đến 24 giờ**

| Field | Detail |
|---|---|
| **Ngày** | 2025-05-20 |
| **Trạng thái** | Accepted |
| **Bối cảnh** | AWS Cost Explorer và Budgets có độ trễ dữ liệu lên đến 24 giờ. Cost guard Lambda không thể phản ứng ngay lập tức với chi phí phát sinh trong ngày. |
| **Quyết định** | Chủ động phòng ngừa: Chạy Lambda quét tài nguyên định kỳ mỗi giờ (Hourly Schedule) thông qua EventBridge, dựa trên trạng thái của thẻ g3:keep=true thay vì chờ dữ liệu chi phí -> Stateful Tagging: Áp dụng thời gian ân hạn (Grace Period) 2 tiếng cho các tài nguyên vi phạm trước khi ép buộc TẮT (Force Stop) -> Chốt chặn cuối (Nuclear Option): Vẫn duy trì cảnh báo AWS Budgets -> SNS. Khi AWS Budgets gửi cảnh báo (dù trễ), Lambda sẽ bỏ qua thời gian ân hạn và tắt lập tức mọi tài nguyên thiếu thẻ giữ máy. |
| **Hệ quả** | **Tích cực**: Một tài nguyên rác (thiếu g3:keep) sẽ chỉ tồn tại tối đa ~3 giờ (1 giờ chờ đến chu kỳ quét + 2 giờ ân hạn) trước khi bị tắt, cô lập thiệt hại tài chính ở mức cực thấp thay vì 24 giờ. **Đánh đổi (Trade-off)**: Phải chấp nhận một lượng overhead siêu nhỏ để chạy Lambda mỗi giờ, và phải tạo ra các thẻ tag quản lý trạng thái tạm thời (CostGuardStatus, FirstDetectedAt) trên tài nguyên. |
| **Giải pháp thay thế đã xem xét** | AWS Config + SSM Auto-remediation

---

## Section 4 — MH-OBS: CloudWatch Observability

### 4.1 Dashboard — Ba loại widget

![image](https://hackmd.io/_uploads/rJ5dcrp1zg.png)
![image](https://hackmd.io/_uploads/SydARth1zg.png)


*Widget titles được dùng:*
- **Custom metric widget:** `AppOrdersCreated` — số đơn hàng tạo mới mỗi phút (namespace `EcommApp/Orders`)
- **Alarm widget:** `HighMemoryUsage-Alarm and RDS Disconnection Alarm`
- **Log Insights widget:** `VPC Log Stream Count and Lambda Status Type Analysis`

---

### 4.2 Code — `PutMetricData` từ ứng dụng

```python
# app/metrics.py
import boto3, time, random, uuid
cloudwatch = boto3.client('cloudwatch', region_name='us-east-1')

print("=== START ===")
for i in range(40): 
    mock_order_id = str(uuid.uuid4())[:8]
    try:
        cloudwatch.put_metric_data(
            Namespace='EcommApp/Orders',
            MetricData=[{
                'MetricName': 'OrdersCreated',
                'Dimensions': [{'Name': 'Environment', 'Value': 'prod'}, {'Name': 'Service', 'Value': 'order-service'}],
                'Value': 1, 'Unit': 'Count', 'Timestamp': time.time()
            }]
        )
        print(f"[{i+1}/40] Order #{mock_order_id}")
    except Exception as e: print(f"Lỗi: {e}")

    
    time.sleep(random.randint(4, 8))

print("FINISHED")
```

---

### 4.3 Alarm Configuration

![image](https://hackmd.io/_uploads/BkhafHhJfl.png)


TRƯỜNG HỢP mem_used_percent > 80

| Field | Value |
|---|---|
| **Alarm name** | `HighMemoryUsage-Alarm` |
| **Metric name** | `mem_used_percent` (namespace `CWAgent`) |
| **Metric type** | EC2 OS-level Memory Monitoring |
| **Monitored instance** | `EC2-BE` |
| **Instance ID** | `i-0c3dc421baedad5be` |
| **Threshold** | `mem_used_percent > 80` |
| **Evaluation period** | 5 minutes |
| **Datapoints to alarm** | 1 of 1 |
| **Statistic** | Average |
| **Period** | 5 minutes |
| **Action destination** | SNS topic `Budget-Alert-Topic` → Gmail notification |
| **Alarm state** | `OK` |
| **Missing data treatment** | Treat missing data as missing |
| **Supporting network service** | VPC Interface Endpoint `com.amazonaws.us-east-1.monitoring` |
| **Purpose** | Monitor EC2 memory usage and trigger notifications when RAM usage exceeds safe threshold |

TRƯỜNG HỢP mem_used_percent > 20
    
![image](https://hackmd.io/_uploads/Hyc-6VpyMl.png)
    
| Field | Value |
|---|---|
| **Alarm name** | `HighMemoryUsage-Alarm` |
| **Metric name** | `mem_used_percent` (namespace `CWAgent`) |
| **Metric type** | EC2 OS-level Memory Monitoring |
| **Monitored instance** | `EC2-BE` |
| **Instance ID** | `i-0c3dc421baedad5be` |
| **Threshold** | `mem_used_percent > 20` |
| **Evaluation period** | 5 minutes |
| **Datapoints to alarm** | 1 of 1 |
| **Statistic** | Average |
| **Period** | 5 minutes |
| **Action destination** | SNS topic `Budget-Alert-Topic` → Gmail notification |
| **Alarm state** | `In alarm` |
| **Current memory usage** | `~21.5%` |
| **Missing data treatment** | Treat missing data as missing |
| **Supporting network service** | VPC Interface Endpoint `com.amazonaws.us-east-1.monitoring` |
| **Purpose** | Test CloudWatch EC2 memory monitoring and SNS email notification when RAM usage exceeds configured threshold |

---
![image](https://hackmd.io/_uploads/BJVPIS6yMe.png)
| Field | Value |
|---|---|
| **Alarm name** | `RDS-Disconnect-Alarm` |
| **Metric name** | `DatabaseConnections` (namespace `AWS/RDS`) |
| **Metric type** | RDS Database Connection Monitoring |
| **Monitored database** | `week6-db` |
| **Threshold** | `DatabaseConnections < 1` |
| **Evaluation period** | 1 minute |
| **Datapoints to alarm** | 1 of 1 |
| **Statistic** | Average |
| **Period** | 1 minute |
| **Current database connections** | `~2 active connections` |
| **Action destination** | SNS topic `RDS-Alert-Topic` → Gmail notification |
| **Alarm state** | `OK` |
| **Missing data treatment** | Treat missing data as bad (breaching threshold) |
| **Purpose** | Monitor active RDS database connections and detect unexpected database disconnects or outages |

![image](https://hackmd.io/_uploads/BJj3UHayMl.png)
| Field | Value |
|---|---|
| **Alarm name** | `RDS-Disconnect-Alarm` |
| **Metric name** | `DatabaseConnections` (namespace `AWS/RDS`) |
| **Metric type** | RDS Database Connection Monitoring |
| **Monitored database** | `week6-db` |
| **Threshold** | `DatabaseConnections < 1` |
| **Evaluation period** | 1 minute |
| **Datapoints to alarm** | 1 of 1 |
| **Statistic** | Average |
| **Period** | 1 minute |
| **Current database connections** | `0 active connections` |
| **Action destination** | SNS topic `RDS-Alert-Topic` → Gmail notification |
| **Alarm state** | `In alarm` | 
| **Missing data treatment** | Treat missing data as bad (breaching threshold) |
| **Purpose** | Detect database disconnect situations and automatically trigger CloudWatch + SNS email notifications when no active RDS connections are detected |
    
### 4.4 Log Insights Query

#### [Track 1] — Truy vấn Phân tích Mật độ Lưu lượng Mạng VPC
**Query text:**

```sql
fields @timestamp, @message
| stats count(*) as total_events by logStream
| sort total_events desc
| limit 10
```
Cơ chế thực thi của cú pháp lệnh (Code Syntax Breakdown):
- fields @timestamp, @message: Chỉ định bộ lọc cột để trích xuất cấu trúc mốc thời gian hệ thống (@timestamp) và chuỗi dữ liệu phi cấu trúc chứa thông tin gói tin mạng (@message) từ tệp nhật ký gốc.
- stats count(*) as total_events by logStream: Thực hiện toán tử tập hợp số lượng bản ghi định kỳ. Tham số by logStream phân nhóm các tập hợp dữ liệu theo định danh của từng giao tiếp mạng ảo (Elastic Network Interface - ENI) để tính toán tổng lưu lượng phân phối trên từng cổng kết nối.
- sort total_events desc: Sắp xếp các giá trị mảng dữ liệu sau khi tập hợp theo thứ tự tuyến tính giảm dần dựa trên biến số total_events.
- limit 10: Ranh giới kỹ thuật nhằm cấu hình giới hạn số lượng bản ghi đầu ra tối đa là 10, tối ưu hóa tốc độ quét (Scan size) và giảm thiểu chi phí tài nguyên xử lý dữ liệu. 
    
**Log group:** `/aws/vpc/app-vpc/flowlogs`  
**Query time range:** 1h  
**Saved query name:** `G3_VPC_Log_Stream_Count`

![image](https://hackmd.io/_uploads/SJkif4nyGx.png)


#### [Track 2] — Truy vấn Phân loại Trạng thái Vòng đời Thực thi Lambda

**Query text:**
```sql
fields @timestamp, @message
| parse @message /(?<log_type>INFO|START|END|REPORT|ERROR)/
| stats count(*) as total_logs by log_type
| sort total_logs desc
```
Cơ chế thực thi của cú pháp lệnh (Code Syntax Breakdown):
- parse @message /(?<log_type>INFO|START|END|REPORT|ERROR)/: Sử dụng biểu thức chính quy (Regular Expression - Regex) để phân rã cấu trúc chuỗi ký tự thô. Lệnh tiến hành bóc tách các mẫu ký tự đặc trưng về phân cấp hệ thống để gán vào biến định danh log_type.
- stats count(*) as total_logs by log_type: Thực thi hàm đếm tổng số lượng sự kiện (total_logs) và phân loại đồng thời theo các giá trị biến đổi của mảng log_type.
- sort total_logs desc: Sắp xếp thứ tự hiển thị của bảng dữ liệu theo mật độ xuất hiện giảm dần của các phân cấp log.
    
**Log groups:**   
    - /aws/lambda/hofang-SecurityGuard-RevokeOpenSSH  
    - /aws/lambda/MH-COST-A-AutoStop  
    - /aws/lambda/w6-mhsec-s3-guard  

**Query time range:** 1 giờ (1h)

**Saved query name:** G3_Lambda_Status_Type_Analysis
![image](https://hackmd.io/_uploads/Sk65ISh1Ml.png)
![image](https://hackmd.io/_uploads/SJxmwrnJzg.png)


---

## Section 5 — MH-SEC: Self-Healing Security Guard

### 5.1 Lambda Code/Config

```python
import boto3
import json

ec2 = boto3.client('ec2')

def lambda_handler(event, context):
    """
    Self-Healing Security Guard
    Detect SG rules opening SSH (port 22) to 0.0.0.0/0 and auto-revoke them.
    """
    print("=== Security Guard Lambda triggered ===")
    print(f"Event: {json.dumps(event, default=str)}")
    
    fixed_count = 0
    
    # Describe all security groups
    response = ec2.describe_security_groups()
    
    for sg in response['SecurityGroups']:
        sg_id = sg['GroupId']
        sg_name = sg['GroupName']
        
        for rule in sg['IpPermissions']:
            # Check if rule opens port 22 (SSH)
            from_port = rule.get('FromPort', 0)
            to_port = rule.get('ToPort', 0)
            
            if from_port <= 22 <= to_port:
                # Check if source is 0.0.0.0/0
                for ip_range in rule.get('IpRanges', []):
                    if ip_range.get('CidrIp') == '0.0.0.0/0':
                        print(f"❌ VIOLATION FOUND: {sg_name} ({sg_id}) has SSH open to 0.0.0.0/0")
                        
                        # Revoke the rule
                        try:
                            ec2.revoke_security_group_ingress(
                                GroupId=sg_id,
                                IpPermissions=[{
                                    'IpProtocol': rule['IpProtocol'],
                                    'FromPort': from_port,
                                    'ToPort': to_port,
                                    'IpRanges': [{'CidrIp': '0.0.0.0/0'}]
                                }]
                            )
                            print(f"✅ FIXED: Revoked SSH 0.0.0.0/0 from {sg_name} ({sg_id})")
                            fixed_count += 1
                        except Exception as e:
                            print(f"⚠️ ERROR revoking {sg_id}: {str(e)}")
                
                # Also check for ::/0 (IPv6)
                for ip_range in rule.get('Ipv6Ranges', []):
                    if ip_range.get('CidrIpv6') == '::/0':
                        print(f"❌ VIOLATION FOUND: {sg_name} ({sg_id}) has SSH open to ::/0 (IPv6)")
                        
                        try:
                            ec2.revoke_security_group_ingress(
                                GroupId=sg_id,
                                IpPermissions=[{
                                    'IpProtocol': rule['IpProtocol'],
                                    'FromPort': from_port,
                                    'ToPort': to_port,
                                    'Ipv6Ranges': [{'CidrIpv6': '::/0'}]
                                }]
                            )
                            print(f"✅ FIXED: Revoked SSH ::/0 from {sg_name} ({sg_id})")
                            fixed_count += 1
                        except Exception as e:
                            print(f"⚠️ ERROR revoking IPv6 {sg_id}: {str(e)}")
    
    result = {
        'statusCode': 200,
        'body': f'Security Guard scan complete. Fixed {fixed_count} violation(s).'
    }
    print(f"=== Result: {result} ===")
    return result

```

**Screenshot — Lambda function:**

![image](https://hackmd.io/_uploads/B1NT5En1zg.png)


---

### 5.2 Least-Privilege IAM Role (Security Guard)

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "AllowDescribeAllSecurityGroups",
			"Effect": "Allow",
			"Action": "ec2:DescribeSecurityGroups",
			"Resource": "*"
		},
		{
			"Sid": "AllowRevokeDevSecurityGroupsOnly",
			"Effect": "Allow",
			"Action": "ec2:RevokeSecurityGroupIngress",
			"Resource": "arn:aws:ec2:*:*:security-group/*",
			"Condition": {
				"StringEquals": {
					"aws:ResourceTag/Environment": [
						"dev",
						"prod"
					]
				}
			}
		},
		{
			"Sid": "AllowCreateSpecificLogGroup",
			"Effect": "Allow",
			"Action": "logs:CreateLogGroup",
			"Resource": "arn:aws:logs:*:*:log-group:/aws/lambda/hofang-SecurityGuard-RevokeOpenSSH"
		},
		{
			"Sid": "AllowWriteToSpecificLogStream",
			"Effect": "Allow",
			"Action": [
				"logs:CreateLogStream",
				"logs:PutLogEvents"
			],
			"Resource": "arn:aws:logs:*:*:log-group:/aws/lambda/hofang-SecurityGuard-RevokeOpenSSH:*"
		}
	]
}
```

**Screenshot — IAM Role:**

![image](https://hackmd.io/_uploads/HyqiFVakze.png)

---

### 5.3 Trigger — EventBridge Rule

![image](https://hackmd.io/_uploads/rylQnN31Ge.png)


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

![image](https://hackmd.io/_uploads/r1FHn4hkze.png)


**After (remediated — rule đã bị revoke):**

![image](https://hackmd.io/_uploads/r11Oh4hyzg.png)


**CloudTrail event — `RevokeSecurityGroupIngress`:**

```json
"eventTime": "2026-05-21T07:02:38Z",
    "eventSource": "ec2.amazonaws.com",
    "eventName": "RevokeSecurityGroupIngress",
    "awsRegion": "us-east-1",
    "sourceIPAddress": "3.88.49.54",
    "userAgent": "Boto3/1.40.4 md/Botocore#1.40.4 ua/2.1 os/linux#5.10.252-285.992.amzn2.x86_64 md/arch#x86_64 lang/python#3.12.13 md/pyimpl#CPython exec-env/AWS_Lambda_python3.12 m/D,b,Z cfg/retry-mode#legacy Botocore/1.40.4",
    "requestParameters": {
        "groupId": "sg-04c4d945debdf43b6",
        "ipPermissions": {
            "items": [
                {
                    "ipProtocol": "tcp",
                    "fromPort": 22,
                    "toPort": 22,
                    "groups": {},
                    "ipRanges": {
                        "items": [
                            {
                                "cidrIp": "0.0.0.0/0"
                            }
                        ]
                    },
                    "ipv6Ranges": {},
                    "prefixListIds": {}
                }
            ]
        }
    },
```

![image](https://hackmd.io/_uploads/By9R3V21Me.png)


---

### 5.5 Supporting Preventive Control

Nhóm chọn `Path B` làm supporting preventive control cho `MH-SEC`: bật `account-level S3 Block Public Access` với đầy đủ `4` setting, và thêm `bucket policy deny PutObject` khi request thiếu `server-side encryption` header. Control này không thay thế `Lambda auto-remediation` của main guard; nó là lớp phòng ngừa bổ sung ở tầng `S3` để chặn `public access` và upload không an toàn ngay từ đầu

---
### What We Implemented

1. Vào `S3 -> Block Public Access settings for this account`.
2. Bật đủ `4/4` setting ở mức account.
3. Vào bucket demo `w6-mhsec-demo-team03-001`.
4. Gắn `bucket policy deny` để chặn `PutObject` khi request không gửi header `s3:x-amz-server-side-encryption`.
5. Dùng `AWS CLI` trên PowerShell để chạy một test call cố ý vi phạm policy và nhận `AccessDenied`.

### Bucket Policy Used
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedObjectUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::w6-mhsec-demo-team03-001/*",
      "Condition": {
        "Null": {
          "s3:x-amz-server-side-encryption": "true"
        }
      }
    }
  ]
}
```

#### Option B — S3 Block Public Access (Account-level) + Deny Policy

![lambda1](https://hackmd.io/_uploads/BJwFqIn1zx.png)
![lambda2](https://hackmd.io/_uploads/S1wt9I21Gg.png)
![lambda3](https://hackmd.io/_uploads/rywt9UnkMl.png)
![lambda4](https://hackmd.io/_uploads/SJDFqU2JMg.png)
![lambda5](https://hackmd.io/_uploads/S1W3cI31Gl.png)







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

Nhóm xây dựng nền tảng **e-commerce** phục vụ thị trường bán lẻ B2C, bao gồm các tính năng: duyệt sản phẩm, quản lý giỏ hàng, đặt hàng và thanh toán trực tuyến. Business domain là **retail commerce**, hướng tới SMB muốn số hoá hoạt động bán hàng mà không cần đầu tư hạ tầng lớn.

### Quyết định kiến trúc và thiết kế chính (W1–W6)

- **W1–W2:** Kiến trúc three-tier (ALB → ECS Fargate → RDS Multi-AZ). Chọn region `ap-southeast-1` (Singapore) để tối ưu latency cho người dùng Đông Nam Á.
- **W3:** Bổ sung database backup strategy (automated snapshots, retention policy) và lựa chọn engine phù hợp với workload e-commerce.
- **W4:** Tích hợp AI layer vào luồng nghiệp vụ chatbot hỏi đáp (RAG)
- **W5:** Triển khai API Gateway làm entry point cho lambda. Bổ sung VPC Network Firewall để kiểm soát traffic ở layer network, tăng cường bảo mật perimeter.
- **W6 (hiện tại):** Thêm lớp vận hành: cost tagging theo project, Cost Anomaly Detection + EventBridge + SNS alerting, CloudWatch Observability dashboard, và self-healing security Lambda.

### W5 Feedback đã giải quyết

| Feedback | Trạng thái | Cách xử lý |
|----------|------------|------------|
| API Key bị lộ nguyên giá trị trong evidence pack | ✅ Đã xử lý | Rotate key |
| Presigned URL chứa STS temp credentials | ✅ Đã xử lý | Redact khỏi pack, xem xét Cognito JWT cho production |


## Bonus Section *(tuỳ chọn)*
### Cost Anomaly Automation

**Kịch bản:** Hệ thống phát hiện chi phí bất thường và tự động cảnh báo qua email khi có spike.

Giả lập tình huống EC2 tại `ap-southeast-1` đột ngột tăng usage bất thường:

| | Giá trị |
|---|---|
| Expected spend | $5.00 (mức bình thường theo lịch sử) |
| Actual spend | $50.50 (tăng 10x) |
| Anomaly score | 99/100 |
| Root cause | `BoxUsage:t3.micro` chạy 100 giờ liên tục |

**Trigger fake anomaly event qua CLI:**

```bash
aws events put-events \
  --entries '[{
    "Source": "custom.costanomalydetection",
    "DetailType": "Cost Anomaly Detection Alert",
    "Detail": "{
      \"anomalyId\": \"fake-anomaly-demo-001\",
      \"anomalyScore\": { \"maxScore\": 99, \"currentScore\": 99 },
      \"impact\": {
        \"maxImpact\": 45.50,
        \"totalImpact\": 45.50,
        \"totalActualSpend\": 50.50,
        \"totalExpectedSpend\": 5.00
      },
      \"monitorName\": \"g3-project-monitor\",
      \"anomalyStartDate\": \"2026-05-22\",
      \"rootCauses\": [{
        \"service\": \"Amazon EC2\",
        \"region\": \"ap-southeast-1\",
        \"usageType\": \"BoxUsage:t3.micro\",
        \"usageAmount\": \"100\"
      }]
    }",
    "EventBusName": "default"
  }]' \
  --region us-east-1
```

**Pipeline hoạt động:**

```
custom.costanomalydetection event
           ↓
  EventBridge Rule: CostAnomalyToSNS
           ↓
  SNS Topic: cost-anomaly-alerts
           ↓
  📧 tuphucnguyen20051@gmail.com (~30 giây)
```

**Kết quả:** Email "AWS Notification Message" nhận được tại inbox với đầy đủ thông tin anomaly — chứng minh pipeline cảnh báo hoạt động end-to-end, đáp ứng yêu cầu proactive alerting khi chi phí vượt ngưỡng bất thường.

![SNS Email Notification](https://hackmd.io/_uploads/r1GY4BaJGe.png)



*— Kết thúc W6 Evidence Pack —*