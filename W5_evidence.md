# W5 Evidence Pack — The Network Fortress

---

## 1. Cover

| Thông tin | Nội dung |
|-----------|----------|
| **Group ID** | `GROUP-XX` |
| **Tuần** | W5 — 11–15 tháng 5, 2026 |
| **Repo** | [https://github.com/your-org/your-repo](https://github.com/your-org/your-repo) |
| **Evidence Pack W4** | [docs/W4_evidence.md](../docs/W4_evidence.md) |

### Thành viên

| # | Họ tên | Nhiệm vụ W5 |
|---|--------|-------------|
| 1 | Nguyễn Văn A | MH1 — Multi-VPC Connectivity |
| 2 | Trần Thị B | MH1 — Multi-VPC Connectivity |
| 3 | Lê Văn C | MH2 — Network Firewall Hardening |
| 4 | Phạm Thị D | MH2 — Network Firewall Hardening |
| 5 | Hoàng Văn E | MH3 — EFS File Storage |
| 6 | Vũ Thị F | MH3 — AWS Backup + Restore Test |
| 7 | Đặng Văn G | MH4 — API Gateway |
| 8 | Bùi Thị H | MH5 — Serverless Scaling Pattern |

---

## 2. MH1 — Multi-VPC Connectivity

### Path đã chọn

- [ ] **Path A** — VPC Peering (2–3 VPC, CIDR không chồng, point-to-point)
- [ ] **Path B** — AWS Transit Gateway (3+ VPC, transitive routing)
- [ ] **Path C** — Justified Single-VPC (có justification rõ ràng)

> **Điền lựa chọn vào đây:** Path X

### Rationale

> _(Giải thích vì sao chọn path này — phải cụ thể cho ứng dụng của nhóm, không phải câu chữ chung chung. Ví dụ: "App tier và AI inference tier cần traffic nội bộ nhưng phải cô lập về security boundary, nên dùng VPC Peering thay vì đặt chung một VPC.")_

**[Viết rationale tại đây]**

---

### Path A/B — Cấu hình kết nối

> _(Điền nếu chọn Path A hoặc B)_

- **VPC nguồn:** `vpc-XXXXXXXX` — CIDR: `10.0.0.0/16`
- **VPC đích:** `vpc-YYYYYYYY` — CIDR: `10.1.0.0/16`
- **Peering Connection ID / TGW ID:** `pcx-XXXXXXXX` / `tgw-XXXXXXXX`

#### Screenshot — Route Table cả hai phía

**Route table VPC nguồn:**

![Route table VPC nguồn](./screenshots/mh1_route_table_src.png)

> _Chú thích: route 10.1.0.0/16 → pcx-XXXXXXXX đã được thêm_

**Route table VPC đích:**

![Route table VPC đích](./screenshots/mh1_route_table_dst.png)

> _Chú thích: route 10.0.0.0/16 → pcx-XXXXXXXX đã được thêm_

#### Test Connectivity Cross-VPC

```bash
# Chạy từ EC2 trong VPC nguồn sang private IP của EC2 trong VPC đích
curl http://10.1.0.XX:8080/health
# Hoặc:
ping 10.1.0.XX
```

**Kết quả:**

```
# Dán output tại đây
```

---

### Path C — Justified Single-VPC

> _(Điền nếu chọn Path C)_

**Justification cụ thể cho ứng dụng:**

> **[Viết tại đây]** — Giải thích vì sao ứng dụng này không cần multi-VPC (dựa trên business logic, không phải lý do tiện lợi).

**Các subnet đã được multi-AZ:**

| Subnet | AZ | CIDR |
|--------|----|------|
| Public subnet A | ap-southeast-1a | `10.0.1.0/24` |
| Public subnet B | ap-southeast-1b | `10.0.2.0/24` |
| Private subnet A | ap-southeast-1a | `10.0.11.0/24` |
| Private subnet B | ap-southeast-1b | `10.0.12.0/24` |

**Event nào sẽ trigger thêm VPC thứ hai:**

> **[Viết tại đây]** — Ví dụ: khi tách môi trường staging hoàn toàn với production, hoặc khi thêm partner integration cần network isolation.

---

### VPC Flow Logs (bắt buộc mọi path)

- **Log group / S3 bucket:** `[điền tên]`
- **VPC đã bật Flow Logs:** `vpc-XXXXXXXX`

#### Screenshot — Flow Logs enabled

![VPC Flow Logs enabled](./screenshots/mh1_flow_logs_enabled.png)

#### Sample Log Entry

```
# Dán một dòng log mẫu từ CloudWatch Logs / S3
# Ví dụ:
2 123456789012 eni-abc123 10.0.1.10 10.1.0.20 443 52314 6 10 840 1715000000 1715000060 ACCEPT OK
```

> _Giải thích: log entry trên cho thấy traffic từ `10.0.1.10` đến `10.1.0.20:443` đã được ACCEPT — xác nhận kết nối cross-VPC hoạt động đúng._

---

## 3. MH2 — Network Firewall Hardening

### Path đã chọn

- [ ] **Path A** — AWS Network Firewall (có NAT Gateway / egress ra internet)
- [ ] **Path B** — Hardened SG + NACL (topology cô lập, không có NAT Gateway)

> **Điền lựa chọn vào đây:** Path X

### Rationale

> **[Viết tại đây]** — Giải thích vì sao chọn path này. Nếu Path B: giải thích (a) vì sao egress firewall không cần thiết và (b) traffic nào sẽ đòi deploy nó trong production.

---

### Path A — AWS Network Firewall

> _(Điền nếu chọn Path A)_

- **Firewall endpoint:** `vpce-XXXXXXXX` — trong subnet `subnet-XXXXXXXX` (Firewall subnet)
- **Stateful rule group:** `[tên rule group]` — loại: Domain-based egress allowlist / IPS signature
- **Alert Logs destination:** CloudWatch log group `[tên]` / S3 bucket `[tên]`

#### Screenshot — Firewall Deployment

![AWS Network Firewall deployed](./screenshots/mh2_firewall_deployed.png)

#### Screenshot — Stateful Rule Group

![Stateful rule group](./screenshots/mh2_stateful_rules.png)

#### Screenshot — Route Table (traffic qua Firewall → NAT)

![Route table with firewall](./screenshots/mh2_route_table_firewall.png)

#### Request được ALLOW (thấy trong Flow Logs)

```
# Dán Flow Log entry của request được cho phép
```

#### Request bị BLOCK (thấy trong Alert Logs)

```
# Dán Alert Log entry của request bị chặn
```

![Alert log blocked request](./screenshots/mh2_alert_log_blocked.png)

---

### Path B — Hardened SG + NACL

> _(Điền nếu chọn Path B)_

**Các thay đổi đã thực hiện:**

- [ ] Đã xóa tất cả rule inbound `0.0.0.0/0` trên port 22/3389 ở mọi Security Group
- [ ] Đã thêm ít nhất một NACL DENY rule rõ ràng
- [ ] Mọi AWS service access đều qua VPC endpoint (không có NAT Gateway)

#### Screenshot — Security Group sau khi hardening

![Security Group hardened](./screenshots/mh2_sg_hardened.png)

> _Chú thích: không còn rule inbound 0.0.0.0/0 trên port 22/3389_

#### Screenshot — NACL DENY Rule

![NACL deny rule](./screenshots/mh2_nacl_deny.png)

#### Negative Test — Kết nối bị từ chối

```bash
# Lệnh test
ssh -i key.pem ec2-user@<IP> -p 22
# Hoặc:
curl http://<IP>:<blocked-port>
```

**Kết quả:**

```
# Dán output lỗi tại đây (timeout / connection refused)
```

![Negative test connection refused](./screenshots/mh2_negative_test.png)

---

## 4. MH3 — File Storage + Backup Plan

### File Storage — Amazon EFS

- **EFS File System ID:** `fs-XXXXXXXX`
- **Mount target subnet:** Private Application Subnet (`subnet-XXXXXXXX`)
- **Security Group mount target:** `sg-XXXXXXXX` — chỉ allow inbound NFS (port 2049) từ `sg-XXXXXXXX` (App tier SG)

#### Screenshot — EFS Console

![EFS file system](./screenshots/mh3_efs_console.png)

#### Screenshot — Security Group Mount Target (chỉ allow từ App SG)

![EFS security group](./screenshots/mh3_efs_sg.png)

#### Bằng chứng mount và ghi/đọc file

```bash
# Mount EFS (chạy trên EC2 trong private subnet)
sudo mount -t efs -o tls fs-XXXXXXXX:/ /mnt/efs

# Ghi file
echo "W5 EFS test - $(date)" | sudo tee /mnt/efs/app/test_$(date +%s).txt

# Đọc lại
cat /mnt/efs/app/test_*.txt
```

**Output:**

```
# Dán output tại đây
```

![EFS write read proof](./screenshots/mh3_efs_write_read.png)

> _Chú thích: file ghi từ instance `i-XXXXXXXX` trong private subnet, đọc lại thành công — xác nhận EFS mount hoạt động đúng._

---

### Backup Plan — AWS Backup

- **Backup Plan Name:** `w5-app-backup-plan`
- **Backup Vault:** `w5-backup-vault`
- **Schedule:** Daily tại `03:00 UTC`
- **Retention:** 7 ngày

**Resources được bao trùm:**

| Resource Type | Resource ID | Ghi chú |
|---------------|-------------|---------|
| EFS | `fs-XXXXXXXX` | File storage W5 |
| RDS / DynamoDB | `[tên DB]` | Database W3 |
| EBS Volume | `vol-XXXXXXXX` | Volume W2 |

#### Screenshot — Backup Plan Configuration

![AWS Backup plan](./screenshots/mh3_backup_plan.png)

#### Screenshot — Backup Vault với Recovery Point

![Backup vault recovery point](./screenshots/mh3_recovery_point.png)

> _Recovery point status: **Completed**_

---

### Restore Test (bắt buộc)

- **Recovery Point ARN:** `arn:aws:backup:ap-southeast-1:123456789012:recovery-point:XXXXXXXX`
- **Restore job ID:** `[job ID]`
- **Thời gian trigger:** `[timestamp]`
- **Thời gian complete:** `[timestamp]`

#### Screenshot — Restore Job Completed

![Restore job completed](./screenshots/mh3_restore_job_completed.png)

> _Chú thích: restore job status **Completed** — resource đã được khôi phục thành công._

#### Kết nối vào resource đã khôi phục và đọc data

```bash
# Kết nối vào resource đã restore (EFS/RDS/EC2 tùy loại)
# Ví dụ với EFS:
sudo mount -t efs -o tls fs-YYYYYYYY:/ /mnt/restored
cat /mnt/restored/app/test_*.txt

# Ví dụ với RDS:
mysql -h restored-db.XXXXXXXX.ap-southeast-1.rds.amazonaws.com -u admin -p
SELECT * FROM your_table LIMIT 5;
```

**Output — data đọc được từ resource đã restore:**

```
# Dán output tại đây — phải khớp với data đã biết trước khi backup
```

![Data read from restored resource](./screenshots/mh3_restore_data_readback.png)

> _Xác nhận: data từ resource đã restore khớp với data gốc → backup plan hoạt động đúng._

---

## 5. MH4 — API Gateway trước Lambda

### Thông tin API

- **API Type:** REST API / HTTP API _(chọn một)_
- **API Gateway ID:** `[api-id]`
- **Stage:** `prod`
- **Invoke URL:** `https://XXXXXXXXXX.execute-api.ap-southeast-1.amazonaws.com/prod`
- **Lambda function phía sau:** `[tên function]`

### Cây Resource (Resource Tree)

```
/
└── /query                    (hoặc endpoint thật của app)
    ├── GET  → Lambda: [tên function]  [Lambda Proxy Integration]
    └── POST → Lambda: [tên function]  [Lambda Proxy Integration]
```

#### Screenshot — API Gateway Resource Tree

![API Gateway resource tree](./screenshots/mh4_resource_tree.png)

---

### Cấu hình Authentication

- **Auth type:** API Key / Lambda Authorizer / Cognito User Pool Authorizer _(chọn một)_
- **Authorizer name / Key name:** `[tên]`

#### Screenshot — Auth Configuration

![API Gateway auth configuration](./screenshots/mh4_auth_config.png)

---

### Throttling — Usage Plan

- **Usage Plan name:** `w5-usage-plan`
- **Rate limit:** `[X] requests/second`
- **Burst limit:** `[Y] requests`

#### Screenshot — Usage Plan & Throttling

![API Gateway throttling](./screenshots/mh4_throttling.png)

---

### Test curl

#### Authenticated Request → 200 OK

```bash
curl -X GET \
  "https://XXXXXXXXXX.execute-api.ap-southeast-1.amazonaws.com/prod/query" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "test"}'
```

**Response:**

```json
HTTP/1.1 200 OK
{
  "statusCode": 200,
  "body": "..."
}
```

![Curl authenticated 200](./screenshots/mh4_curl_200.png)

---

#### Unauthenticated Request → 403 Forbidden

```bash
curl -X GET \
  "https://XXXXXXXXXX.execute-api.ap-southeast-1.amazonaws.com/prod/query" \
  -H "Content-Type: application/json" \
  -d '{"query": "test"}'
```

**Response:**

```json
HTTP/1.1 403 Forbidden
{
  "message": "Forbidden"
}
```

![Curl unauthenticated 403](./screenshots/mh4_curl_403.png)

---

### Xác nhận code app đã cập nhật

> **[Ghi chú]** — Đường dẫn file đã sửa và mô tả thay đổi:
>
> Ví dụ: `src/services/ai_service.py` — thay `lambda_client.invoke(FunctionName=...)` bằng `requests.post("https://XXXXXXXXXX.execute-api.ap-southeast-1.amazonaws.com/prod/query", headers={"x-api-key": ...})`

---

## 6. MH5 — Serverless Scaling Pattern

### Pattern đã chọn

- [ ] **Reserved Concurrency** — giới hạn concurrency tối đa cho một function
- [ ] **Provisioned Concurrency** — pre-warm để loại bỏ cold start
- [ ] **Async Invocation + Dead Letter Queue** — gọi async, lỗi rơi vào DLQ
- [ ] **S3-Event-Triggered Lambda** — S3 PutObject trigger Lambda → DynamoDB

> **Điền lựa chọn vào đây:** Pattern X

### Lambda function áp dụng

- **Function name:** `[tên function thật trong app]`
- **Runtime:** Python 3.12 / Node.js 20.x / ...
- **Vai trò trong app:** `[mô tả ngắn — ví dụ: xử lý Bedrock query từ API Gateway]`

---

### Reserved Concurrency

> _(Điền nếu chọn pattern này)_

- **Reserved concurrency đặt:** `[X]` concurrent executions
- **Lý do chọn limit này:** `[giải thích]`

#### Screenshot — Reserved Concurrency Configuration

![Reserved concurrency config](./screenshots/mh5_reserved_concurrency.png)

#### Bằng chứng Throttle behavior

```bash
# Invoke nhiều hơn limit đồng thời
for i in {1..20}; do
  aws lambda invoke --function-name [tên] --payload '{}' /tmp/out_$i.json &
done
```

![CloudWatch Throttles metric](./screenshots/mh5_throttle_metric.png)

> _CloudWatch metric `Throttles` tăng lên khi vượt quá reserved concurrency limit._

---

### Provisioned Concurrency

> _(Điền nếu chọn pattern này)_

- **Provisioned concurrency đặt:** `[X]` concurrent executions
- **Version / Alias:** `[tên alias]`
- **Chi phí ước tính:** `$[X]/tháng` với cấu hình hiện tại

#### So sánh Cold Start Before / After

**Before (cold start — có Init Duration):**

![Cold start trace before](./screenshots/mh5_cold_start_before.png)

> _Init duration: ~XXXms_

**After (provisioned — Init Duration = 0ms):**

![No cold start after provisioned](./screenshots/mh5_cold_start_after.png)

> _Init duration: 0ms — xác nhận provisioned concurrency loại bỏ cold start._

---

### Async Invocation + Dead Letter Queue

> _(Điền nếu chọn pattern này)_

- **Invocation type:** `Event` (async)
- **DLQ type:** SQS / SNS
- **DLQ ARN:** `arn:aws:sqs:ap-southeast-1:123456789012:[queue-name]`

#### Screenshot — DLQ Configuration trên Lambda

![DLQ configuration](./screenshots/mh5_dlq_config.png)

#### Demo — Invocation thất bại rơi vào DLQ

```bash
# Trigger invocation với payload gây lỗi
aws lambda invoke \
  --function-name [tên] \
  --invocation-type Event \
  --payload '{"trigger_error": true}' \
  /dev/null
```

**Message trong DLQ (kèm chi tiết lỗi):**

```json
{
  "requestId": "XXXXXXXX",
  "errorMessage": "[chi tiết lỗi]",
  "errorType": "...",
  "functionArn": "arn:aws:lambda:..."
}
```

![DLQ message with error detail](./screenshots/mh5_dlq_message.png)

---

### S3-Event-Triggered Lambda

> _(Điền nếu chọn pattern này)_

- **S3 bucket trigger:** `[tên bucket]` — prefix: `[prefix/]`
- **Event type:** `s3:ObjectCreated:Put`
- **DynamoDB output table:** `[tên table]`

#### Screenshot — S3 Event Notification Configuration

![S3 event notification](./screenshots/mh5_s3_event_config.png)

#### Flow End-to-End

**Bước 1 — Thả file vào S3:**

```bash
aws s3 cp test_file.json s3://[bucket]/[prefix]/test_file.json
```

**Bước 2 — CloudWatch Log của Lambda:**

![Lambda CloudWatch log](./screenshots/mh5_lambda_log.png)

**Bước 3 — Output row trong DynamoDB:**

![DynamoDB output row](./screenshots/mh5_dynamodb_output.png)

> _Xác nhận flow end-to-end: file đưa vào S3 → Lambda trigger → data ghi vào DynamoDB._

---

## 7. Application Carry-Forward Verification

> Xác nhận ứng dụng W1–W4 vẫn chạy đúng sau khi deploy lại trên account mới.

### App chạy end-to-end

**Action đại diện thực hiện:** `[mô tả — ví dụ: gửi câu hỏi qua API, nhận câu trả lời từ Bedrock]`

![App end-to-end action](./screenshots/carry_forward_app_running.png)

---

### Bedrock / AI Retrieval

```bash
# Request tới endpoint AI của app
curl -X POST https://[your-api]/query \
  -H "Content-Type: application/json" \
  -d '{"question": "What is the purpose of this application?"}'
```

**Response:**

```json
{
  "answer": "..."
}
```

![Bedrock retrieval working](./screenshots/carry_forward_bedrock.png)

---

### Database Query

```bash
# Ví dụ với RDS / DynamoDB
mysql -h [endpoint] -u admin -p -e "SELECT COUNT(*) FROM your_table;"
# Hoặc:
aws dynamodb scan --table-name [table] --select COUNT
```

**Output:**

```
# Dán output tại đây
```

![Database query result](./screenshots/carry_forward_database.png)

---

### Feedback từ W4 đã được giải quyết

**Feedback cụ thể từ trainer W4:**

> _"[Dán feedback nguyên văn tại đây]"_

**Cách W5 build tiếp / fix:**

> **[Viết tại đây]** — Mô tả cụ thể bước đã làm để giải quyết feedback đó trong W5.

![W4 feedback addressed](./screenshots/carry_forward_feedback.png)

---

## 8. Negative Security Tests

> Ít nhất một negative test cho mỗi bổ sung W5.

### NST-1 — MH1: Traffic không đi được ngoài route đã định

**Mô tả test:** Thử kết nối từ VPC A sang một CIDR không có trong route table → bị timeout.

```bash
# Từ EC2 trong VPC A, thử reach IP không thuộc VPC B đã peer
curl --connect-timeout 5 http://172.16.0.1/
```

**Kết quả mong đợi:** Connection timeout — không có route.

**Kết quả thực tế:**

```
curl: (28) Connection timed out after 5001 milliseconds
```

![NST-1 no unintended route](./screenshots/nst1_no_route.png)

---

### NST-2 — MH2: Request bị Firewall/NACL chặn

**Mô tả test:** Thử kết nối đến domain/IP nằm ngoài allowlist hoặc bị NACL DENY.

```bash
# Path A: domain bị firewall chặn
curl --connect-timeout 5 http://blocked-domain.example.com/

# Path B: port bị NACL DENY
curl --connect-timeout 5 http://[internal-ip]:22
```

**Kết quả mong đợi:** Connection refused / timeout — firewall hoặc NACL block.

**Kết quả thực tế:**

```
# Dán output tại đây
```

![NST-2 firewall block](./screenshots/nst2_firewall_block.png)

---

### NST-3 — MH3: EFS không accessible từ ngoài App SG

**Mô tả test:** Thử mount EFS từ EC2 không thuộc App tier Security Group → bị từ chối.

```bash
# Từ EC2 không có trong App SG
sudo mount -t efs -o tls fs-XXXXXXXX:/ /mnt/test_efs
```

**Kết quả mong đợi:** Mount thất bại — SG của mount target không cho phép.

**Kết quả thực tế:**

```
mount.nfs4: Connection timed out
```

![NST-3 EFS unauthorized mount](./screenshots/nst3_efs_block.png)

---

### NST-4 — MH4: API Gateway từ chối request không có auth

**Mô tả test:** Gọi endpoint API Gateway không kèm API Key / token.

```bash
curl -X GET \
  "https://XXXXXXXXXX.execute-api.ap-southeast-1.amazonaws.com/prod/query"
```

**Kết quả mong đợi:** `403 Forbidden`

**Kết quả thực tế:**

```json
HTTP/1.1 403 Forbidden
{"message": "Forbidden"}
```

![NST-4 API Gateway 403](./screenshots/mh4_curl_403.png)

> _Đây là negative test cho MH4 — đã được document ở mục MH4 ở trên._

---

### NST-5 — MH5: Lambda bị throttle khi vượt concurrency limit

**Mô tả test:** Invoke Lambda vượt quá reserved concurrency → nhận TooManyRequestsException.

```bash
# Invoke đồng thời nhiều hơn reserved concurrency limit
for i in {1..30}; do
  aws lambda invoke \
    --function-name [tên] \
    --payload '{}' \
    /tmp/out_$i.json 2>&1 &
done
wait
grep -l "TooManyRequests" /tmp/out_*.json | wc -l
```

**Kết quả mong đợi:** Một số invocation nhận `TooManyRequestsException` hoặc metric `Throttles` > 0.

**Kết quả thực tế:**

```
# Dán output tại đây
```

![NST-5 Lambda throttle](./screenshots/nst5_lambda_throttle.png)

---

*Evidence Pack W5 — Group XX · 15/05/2026*
