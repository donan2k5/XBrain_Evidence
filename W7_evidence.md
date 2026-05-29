
---

# W7 Evidence Pack — AI Study Buddy

---

## Section 1 — Cover

| Thông tin | Chi tiết |
|-----------|----------|
| **Số nhóm** | Nhóm 3 |
| **Tên project** | AI Study Buddy |
| **Domain** | Topic 1 — EduTech |
| **Thành viên** | Đoàn Văn An <br> Ngô Thanh Kiên <br> Nguyễn Minh Thanh <br> Nguyễn Thành Đạt <br> Đinh Văn Ty <br> Phan Lê Thanh Hoàng <br> Hoàng Minh Hải <br> Từ Phúc Nguyên <br> Nguyễn Văn Toàn <br> Lê Trần Ánh Nhung |
| **Live Public URL HTTPS** | `https://d3ozf26w03ege8.cloudfront.net/` |
| **GitHub Repo** |`https://github.com/2hm1901/eduTechG3New` |
| **Commit Evidence Pack** | `https://github.com/donan2k5/XBrain_Evidence/blob/main/W7_evidence.md` |


---

## Section 2 — Pitch & Vision

### Use Case

AI Study Buddy là hệ thống nội bộ do nhà trường triển khai nhằm hỗ trợ sinh viên ôn thi. Hệ thống cho phép sinh viên upload hoặc truy cập các tài liệu học tập, bài giảng lưu hành nội bộ dạng PDF/slide. AI sẽ tự động phân tích nội dung, tóm tắt bài học, trích xuất các topic quan trọng, tạo quiz và hỗ trợ hỏi đáp bám sát 100% vào giáo trình gốc của trường.

### Target Users

* Sinh viên cần ôn tập nhanh trước bài kiểm tra định kỳ hoặc thi học kỳ.
* Người tự học muốn biến tài liệu dài thành summary, topic và quiz một cách hệ thống.
* Khách hàng B2B (Các trường Đại học/Học viện): Cần một nền tảng AI giáo dục riêng tư, không làm rò rỉ chất xám và tài sản trí tuệ (IP) ra các AI công cộng bên ngoài.

### Problem

Khi học với tài liệu PDF hoặc slide, người học và nhà trường thường gặp các vấn đề:
* Từ phía sinh viên: Tài liệu quá dài, nhiều slide nên khó xác định trọng tâm ôn thi. Việc tự viết summary hoặc tạo câu hỏi tốn rất nhiều thời gian. Chatbot bên ngoài (như ChatGPT) trả lời chung chung, không bám sát giáo trình nội bộ và không có nguồn tham chiếu (citation) để kiểm chứng.
* Từ phía nhà trường (Bảo mật): Các bài giảng, tài liệu chuyên ngành là tài sản trí tuệ lưu hành nội bộ. Nhà trường không muốn rủi ro rò rỉ dữ liệu khi sinh viên tự ý upload lên các tool AI trôi nổi trên Internet.

### Solution

AI Study Buddy giải quyết trọn vẹn cả hai bài toán trên thông qua một workflow học tập khép kín:

* Về tính năng: Tự động tóm tắt nội dung, trích xuất topic trọng tâm, tạo quiz luyện tập và trả lời câu hỏi RAG có trích dẫn nguồn rõ ràng.
* Về bảo mật hạ tầng (Security-First Design): Để đáp ứng yêu cầu khắt khe về bảo mật tài liệu của nhà trường, toàn bộ khối xử lý tính toán (AWS Lambda) và cơ sở dữ liệu được đặt hoàn toàn trong Private Subnet. Hệ thống bị cô lập khỏi Internet công cộng, không có đường ra (No IGW/NAT) và chỉ giao tiếp với AI (Amazon Bedrock) thông qua đường ống nội bộ (VPC Endpoints). Điều này đảm bảo dữ liệu bài giảng không bao giờ đi ngang qua Internet mở, bảo vệ tuyệt đối bản quyền tài liệu của trường.

### Real-world Parallel

AI Study Buddy lấy cảm hứng từ một số sản phẩm AI học tập và document assistant thực tế:
- Google NotebookLM: hỏi đáp và tổng hợp thông tin dựa trên tài liệu người dùng cung cấp.
- Quizlet AI: hỗ trợ tạo nội dung ôn tập và quiz.
- Coursera Coach: hỗ trợ giải thích nội dung học theo từng chủ đề;...

---
## Section 3 — Architecture Overview
### 3.1. System Diagram
![W7-Hackathon (2)](https://hackmd.io/_uploads/rJxjDwIlMl.png)


### 3.2. Data flow
1. Luồng Ingestion & Summarize (Chạy 1 lần): - User upload PDF qua API Gateway -> Lambda lưu file vào S3 PDF -> Trigger Lambda (Extraction) dùng pypdf trích xuất text -> Lưu file .txt và .metadata.json vào S3 Source -> Gọi Bedrock Ingestion Job đưa vào Vector DB -> Gọi Bedrock Nova Lite sinh 5 topics + Summary -> Lưu vào DynamoDB.
2. Luồng Retrieval & Chat (On-demand): User đặt câu hỏi -> API Gateway -> Lambda (Chat) -> Gọi bedrock-agent-runtime:RetrieveAndGenerate với bộ lọc doc_id và user_id -> AI sinh câu trả lời kèm citation -> Trả về Frontend.

### 3.3. Main API Routes
POST /upload: Cấp S3 Pre-signed URL để upload an toàn.

GET /documents: Trích xuất danh sách tài liệu và summary từ DynamoDB cho màn hình Dashboard.

POST /chat: Gửi câu hỏi prompt và nhận câu trả lời RAG có trích dẫn.

GET /dashboard: để truy xuất dữ liệu tổng quan về trạng thái người dùng (User state), tiến độ học tập (Learning progress) và lịch sử hoạt động từ bảng DynamoDB.

POST /sumarize, quiz: để kích hoạt LLM tự động sinh ra bản tóm tắt, trích xuất topic và tạo bộ câu hỏi trắc nghiệm.

---
## Section 4 — Cost Discipline
Ngày 1:
![image](https://hackmd.io/_uploads/S1ARkrUlGg.png)

Ngày 2:
![image](https://hackmd.io/_uploads/ry0WxrLlzl.png)

---
## Section 5 — Security

### 5.1. IAM least privilege
Các Lambda Function không dùng quyền Admin. Ví dụ: Role của Lambda Chat chỉ có đúng các action bedrock:RetrieveAndGenerate, dynamodb:PutItem, dynamodb:Query trên các ARN cụ thể.
![image](https://hackmd.io/_uploads/S1DogSLgGx.png)


### 5.2. Cognito / user identity
Dùng JWT Token từ Amazon Cognito. API Gateway cấu hình Authorizer kiểm tra token hợp lệ trước khi cho phép kích hoạt Lambda.
![image](https://hackmd.io/_uploads/ByJuovUxGl.png)

### 5.3. KMS / encryption
Dữ liệu tĩnh trên S3 (các file PDF bài giảng) được mã hóa.
 ![image](https://hackmd.io/_uploads/HkIj0D8xfe.png)

### 5.4. Network security
Security Groups (SG): Lambda-SG không mở cổng Inbound nào, chỉ cho phép Outbound tới Bedrock-Endpoint-SG qua port 443. Đảm bảo hệ thống kín hoàn toàn từ Internet. Hiện tại, mức độ rủi ro và quy mô hệ thống chưa đủ để justify chi phí vận hành WAF/Network Firewall. Vì vậy, em ưu tiên Security Group, NACL, IAM least privilege, logging/monitoring và hardening trước. Khi hệ thống lớn hơn hoặc có yêu cầu bảo mật rõ ràng hơn, em sẽ cân nhắc bổ sung WAF/Network Firewall.
![image](https://hackmd.io/_uploads/rk4mk_Llfg.png)
![image](https://hackmd.io/_uploads/Skwfyu8xfx.png)

---

## Section 6 — Monitoring / Observability
### 6.1. CloudWatch dashboard
![image](https://hackmd.io/_uploads/ry5PwBLxfx.png)
#### 6.1.1. API Gateway Dashboard

Dashboard này dùng để quan sát traffic đi vào hệ thống qua API Gateway.

Các metric chính:

| Metric | Đọc được gì |
|---|---|
| `Count` | Có bao nhiêu request gọi vào API Gateway |
| `Latency` | API phản hồi nhanh hay chậm |
| `4XXError` | Lỗi phía client như sai request, sai auth, sai path |
| `5XXError` | Lỗi phía server/backend như Lambda lỗi, timeout, integration lỗi |

Nhìn dashboard này để biết:

- API có traffic hay không
- Request có bị lỗi không
- Lỗi đến từ client hay backend
- API Gateway có đang phản hồi chậm bất thường không

---

#### 6.1.2. Lambda Dashboard

Dashboard này dùng để quan sát các Lambda function xử lý backend.

Các metric chính:

| Metric | Đọc được gì |
|---|---|
| `Invocations` | Lambda được gọi bao nhiêu lần |
| `Duration` | Lambda chạy mất bao lâu |
| `Errors` | Lambda có bị lỗi bao nhiêu lần |
| `Throttles` | Lambda có bị giới hạn concurrency không |

Nhìn dashboard này để biết:

- Function nào đang được gọi
- Function nào xử lý chậm
- Function nào đang lỗi
- Có bị throttle do thiếu concurrency không

---

#### 6.1.3. Amazon Bedrock Dashboard

Dashboard này dùng để quan sát việc gọi AI model qua Amazon Bedrock.

Các metric chính:

| Metric | Đọc được gì |
|---|---|
| `Invocations` | Model được gọi bao nhiêu lần |
| `InputTokenCount` | Prompt/input gửi vào model dùng bao nhiêu token |
| `OutputTokenCount` | Model sinh ra bao nhiêu token |
| `InvocationLatency` | Model phản hồi mất bao lâu |
| `Throttles` | Có bị giới hạn quota/rate limit không |

Nhìn dashboard này để biết:

- Model có đang được gọi không
- Mỗi request tốn nhiều token hay ít token
- Bedrock có phản hồi chậm không
- Có nguy cơ tăng cost do token cao không
- Có bị throttle do vượt quota không

---

#### 6.1.4. DynamoDB Dashboard

Dashboard này dùng để quan sát mức đọc/ghi của DynamoDB.

Các metric chính:

| Metric | Đọc được gì |
|---|---|
| `ConsumedReadCapacityUnits` | DynamoDB đang tiêu thụ bao nhiêu read capacity |
| `ConsumedWriteCapacityUnits` | DynamoDB đang tiêu thụ bao nhiêu write capacity |

Nhìn dashboard này để biết:

- Table có đang được đọc/ghi không
- Tải đọc/ghi cao hay thấp
- Có spike bất thường trong read/write không
- DynamoDB có đang idle hay đang được sử dụng nhiều

#### 6.1.5. Custom Dashboard
![image](https://hackmd.io/_uploads/SyXrSUIlMe.png)
Dashboard `ai-study-buddy-finops-ux` được thiết kế chuyên sâu với 4 widget theo dõi thời gian thực (Real-time):
- **AI Total Tokens (Line Chart - Trên trái):** Theo dõi xu hướng tiêu thụ token, bóc tách chi tiết giữa `InputTokens` và `TotalTokens` để kiểm soát chi phí đầu vào.
- **PDF Extraction Latency (Line Chart - Trên phải):** Giám sát độ trễ của luồng bóc tách PDF, được phân loại theo Dimension trạng thái (`success` hoặc `empty_text`) để phát hiện sớm các file lỗi.
- **AI Total Tokens by Operation (Stacked Area - Dưới trái):** Biểu đồ vùng trực quan hóa chính xác tính năng nào đang tiêu tốn nhiều tài nguyên LLM nhất.
- **Recent Custom Metric Logs (Log Insights - Dưới phải):** Stream trực tiếp các dòng log hệ thống chứa payload metric, hỗ trợ debug ngay lập tức.
### 6.2. Alarms
![image](https://hackmd.io/_uploads/HJjnUrLxzg.png)

Dashboard này hiển thị các alarm gần đây của hệ thống. Hiện tại có spike ở `Errors > 0`, chủ yếu liên quan đến `text-extraction`. Các alarm về `Throttles`, `DynamoDB SystemErrors` đều bằng `0`, nghĩa là chưa thấy bị giới hạn concurrency hoặc lỗi hệ thống từ DynamoDB.

Các duration của `dashboard`, `upload-handler`, và `text-extraction` vẫn nằm dưới ngưỡng cảnh báo. Nhìn chung hệ thống chưa có dấu hiệu quá tải, nhưng cần chú ý lỗi phát sinh ở luồng xử lý text extraction.
### 6.3. Log Insights queries
![image](https://hackmd.io/_uploads/HktUcBUeMl.png)
````md
### Query kiểm tra Lambda duration và memory

```sql
fields @timestamp, @requestId, @duration, @billedDuration, @maxMemoryUsed
| filter @type = "REPORT"
| sort @duration desc
| limit 50
````
### 6.5. Decision Log & Measurement
#### 6.5.1. Scenario 1: AI token tăng mạnh
**Measurement**:

TotalTokens tăng từ ~2K lên ~50K trong 5 phút.
InputTokens chiếm 85-90% tổng token.
Operation tốn nhiều nhất: document_chat.

**Insight**:
Chi phí AI tăng chủ yếu vì context gửi vào model quá dài, không     phải vì model trả lời quá dài.
Retrieval có thể đang đưa quá nhiều chunks hoặc chunks quá lớn.

**Decision**:

  Giảm top_k retrieval từ 5 xuống 3.
  Giảm max chunk size hoặc rút ngắn context trước khi gửi vào model.
  Set alarm TotalTokens > 50K / 5 phút.
  Theo dõi lại nếu answer quality giảm.

TRADE-OFF ACCEPTED: Đánh đổi một lượng cực nhỏ dung lượng lưu trữ DynamoDB để lấy hiệu năng (Latency) và tiết kiệm chi phí tính toán AI (Compute).

#### 6.5.2. Scenario 2: PDF extraction latency tăng

**Measurement**:

  PDFExtractionLatency trung bình tăng từ ~1s lên 12s.
  Status vẫn là success.

  **Insight**:

  Bottleneck nằm ở bước bóc PDF bằng pypdf.
  Có thể file lớn, nhiều trang, hoặc PDF phức tạp.

  **Decision**:

  Tăng Lambda memory cho text_extraction từ 1024MB lên 1536/2048MB.
  Giới hạn file size/page count.
  Đưa extraction sang async flow rõ ràng hơn cho UX.
  Set alarm Average PDFExtractionLatency > 10s trong 5 phút.




---
## Section 7 — Trade-offs (Architectural Comparisons)

Dưới đây là các đánh đổi kiến trúc (Trade-offs) mà nhóm đã thảo luận và quyết định khi lựa chọn dịch vụ cho 7 Mandatory Capabilities:

### 7.1. User-Facing Entry
* **Lựa chọn:** Amazon S3 (Static Hosting) + CloudFront.
* **Phương án loại bỏ:** AWS Amplify Hosting | Application Load Balancer (ALB) + EC2.
* **Trade-off:** Amplify giúp CI/CD tự động và setup nhanh hơn rất nhiều, nhưng S3 + CloudFront cho phép nhóm can thiệp sâu vào cấu hình Origin, Caching và Security Headers. Nhóm đánh đổi thời gian setup thủ công ban đầu để lấy sự hiểu biết sâu sắc về CDN và kiểm soát chi phí truyền tải ở mức độ byte. ALB+EC2 bị loại bỏ hoàn toàn vì chi phí cố định quá cao cho một frontend tĩnh.

### 7.2. Application Compute
* **Lựa chọn:** Amazon API Gateway + AWS Lambda.
* **Phương án loại bỏ:** Amazon ECS (Fargate) | Amazon EC2.
* **Trade-off:** AWS Lambda bị giới hạn thời gian chạy tối đa (15 phút) và gặp hiện tượng "Cold Start" (trễ nhịp ở request đầu tiên). Đổi lại, nhóm giải quyết được triệt để bài toán chi phí: Serverless scale về 0 đồng khi không có request, không cần cấu hình Auto Scaling Group và hoàn toàn không tốn công sức quản lý vá lỗi hệ điều hành (OS patching) như EC2.

### 7.3. AI / ML Feature
* **Lựa chọn:** Amazon Bedrock (Knowledge Base với S3 Vectors + model Nova Lite ).
* **Phương án loại bỏ:** Tự code RAG bằng LangChain + OpenSearch Serverless.
* **Trade-off:** Dùng OpenSearch Serverless mang lại tốc độ tìm kiếm vector cực nhanh và hỗ trợ query phức tạp, nhưng có chi phí duy trì cố định cực lớn (tối thiểu 2 OCU tốn ~$27.65/48h). Nhóm đánh đổi việc truy vấn có thể chậm hơn vài chục mili-giây bằng cách dùng **S3 Vectors** để đưa chi phí vector store về mức gần $0. Về LLM, nhóm dùng Nova Lite và Claude Haiku thay vì Claude Sonnet để tối ưu chi phí token (Sonnet đắt hơn gấp ~3 đến 15 lần).

### 7.4. Data Persistence
* **Lựa chọn:** Amazon DynamoDB.
* **Phương án loại bỏ:** Amazon RDS (PostgreSQL).
* **Trade-off:** DynamoDB (NoSQL) không hỗ trợ các câu lệnh `JOIN` phức tạp, đòi hỏi nhóm phải thiết kế Access Pattern (PK, SK) rất chuẩn ngay từ đầu. Tuy nhiên, nhóm chấp nhận đánh đổi sự linh hoạt của SQL để giải quyết "nút thắt cổ chai" của Serverless: Lambda khi scale out sẽ tạo ra hàng nghìn connection làm sập RDS nếu không có RDS Proxy (tốn thêm phí). DynamoDB truy xuất qua HTTP API nên hoàn toàn miễn nhiễm với lỗi connection pool và phí tính theo on-demand cực rẻ.

### 7.5. Object Storage
* **Lựa chọn:** Amazon S3.
* **Phương án loại bỏ:** Lưu trực tiếp base64 vào Database / EFS (Elastic File System).
* **Trade-off:** Việc lưu file PDF lên S3 yêu cầu phải code thêm luồng sinh Pre-signed URL ở API Gateway để user upload gián tiếp, làm tăng độ phức tạp của luồng Frontend-Backend. Nhưng đổi lại, S3 cung cấp nơi lưu trữ vô hạn, chi phí siêu rẻ, và đặc biệt là khả năng kết nối trực tiếp với Bedrock Knowledge Base (điều mà EFS hay Database không làm được).

### 7.6. Network Foundation
* **Lựa chọn:** VPC Private Subnet kết hợp VPC Endpoints (Gateway & Interface).
* **Phương án loại bỏ:** Dùng NAT Gateway để Lambda ra Internet.
* **Trade-off:** Việc loại bỏ NAT Gateway giúp nhóm tiết kiệm ngay ~$1.41/ngày phí duy trì tĩnh. Sự đánh đổi ở đây là rất khắc nghiệt: Lambda của nhóm hoàn toàn "mù" Internet, không thể gọi bất kỳ API 3rd-party nào bên ngoài AWS. Tuy nhiên, vì toàn bộ flow EduTech đều gọi nội bộ (S3, DynamoDB, Bedrock), kiến trúc VPC Endpoints vừa đủ đáp ứng nhu cầu, vừa đẩy độ bảo mật mạng lên mức tối đa tuyệt đối.

### 7.7. Identity & Access
* **Lựa chọn:** Amazon Cognito.
* **Phương án loại bỏ:** Hardcode Test User / Quản lý user bằng bảng tự code trong DB.
* **Trade-off:** Việc thiết lập User Pool, App Client và code logic xác thực JWT ở tầng API Gateway tốn của nhóm khá nhiều thời gian (khoảng 2-3 giờ) so với việc chỉ dùng 1 user hardcode cho lúc Demo. Sự đánh đổi thời gian này mang lại giá trị dài hạn: Hệ thống EduTech có nền tảng Multi-tenant thực sự, giúp phân tách rạch ròi dữ liệu (tenant isolation) giữa các sinh viên khác nhau một cách an toàn.


---

## Section 8 — Reflection (Bài học kinh nghiệm)

- **Điểm mạnh (What went well):** Nhóm kiểm soát rất tốt ngân sách và phân tách rạch ròi các luồng xử lý bằng kiến trúc Event-Driven (S3 Trigger). Việc thống nhất "API Contract" từ ngày đầu tiên giúp Frontend và Backend có thể làm việc song song mà không bị block lẫn nhau.

- **Khó khăn & Hạn chế (Challenges & Weaknesses):**
  - **Giới hạn của thư viện trích xuất:** Thư viện `pypdf` mã nguồn mở xử lý khá tệ các file bài giảng có nhiều bảng biểu phức tạp hoặc phương trình toán học (chữ bị dính vào nhau), làm giảm chất lượng đầu vào của RAG.
  - **Quản lý Source Code & Terraform State:** Đây là "điểm đau" lớn nhất của nhóm. Do chưa có kinh nghiệm setup môi trường multi-developer chuẩn, nhóm quản lý Terraform state chưa tốt (state bị phân mảnh ở máy local của từng người). Hậu quả là khi các thành viên đồng loạt gõ lệnh `terraform apply` hoặc push code, hạ tầng liên tục bị ghi đè (overwrite), dẫn đến việc tính năng của người này vừa chạy được thì bị người kia deploy làm hỏng.

- **Hướng cải thiện (Future Improvements):**
  - **Về mặt tính năng:** Sẽ thêm một lớp logic fallback sang Amazon Textract (mất phí) để bóc tách riêng các slide có tỷ lệ ảnh/bảng biểu cao, giúp RAG trả lời chính xác hơn.
  - **Về mặt quy trình (DevOps/IaC):** Nếu có thêm thời gian, nhóm BẮT BUỘC phải chuyển đổi quản lý Terraform sang mô hình **Remote State** (sử dụng backend S3 kết hợp với bảng DynamoDB để thực hiện State Locking). Đồng thời, thiết lập một luồng CI/CD cơ bản (ví dụ: GitHub Actions) để mọi lệnh deploy đều phải chạy tập trung qua pipeline, chấm dứt hoàn toàn tình trạng "dẫm chân lên nhau" khi deploy từ máy cá nhân.


---
## Section 9 — Teardown Plan

Nhóm ứng dụng **Terraform (Infrastructure as Code - IaC)** để quản lý toàn bộ vòng đời hạ tầng. Do đó, quá trình dọn dẹp sẽ được tự động hóa. Tuy nhiên, để đảm bảo không phát sinh lỗi tài nguyên treo (dangling resources), nhóm cam kết thực hiện theo quy trình 3 bước sau khi kết thúc Hackathon:

1. **Xử lý Dữ liệu tồn đọng (State Clearance):**
   - Các S3 Buckets (S3 PDF, S3 Source, S3 FE) đã được cấu hình `force_destroy = true` trong mã nguồn Terraform để tự động ép xóa dữ liệu bên trong.
   - (Hoặc) Nhóm sẽ chạy script/làm thủ công thao tác "Empty Bucket" trên AWS Console trước để dọn sạch file PDF, JSON metadata và Vector chunks.
2. **Thực thi Hủy Tự Động (Automated Destruction):**
   - Chạy lệnh `terraform destroy` tại terminal quản trị.
   - Terraform sẽ tự động tính toán dependency graph để xóa đúng thứ tự (VD: Xóa VPC Endpoints -> Xóa Private Subnets -> Xóa VPC, xóa Bedrock KB -> S3).
3. **Xác nhận & Báo cáo (Verification):**
   - Truy cập lại AWS Management Console, kiểm tra thủ công màn hình Amazon Bedrock (Knowledge Base), Amazon S3 và VPC để đảm bảo không còn tài nguyên rác.

