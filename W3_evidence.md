# W3 Evidence Pack — Xương Sống Database

<!-- EDIT: Đổi số nhóm, tên thành viên, engine/paradigm, link W2 commit -->

| Thông tin | Giá trị |
|-----------|---------|
| **Nhóm** | Nhóm 3 |
| **Thành viên** | Đoàn Văn An <br> Ngô Thanh Kiên <br> Nguyễn Minh Thanh <br> Lê Trần Ánh Nhung <br> Nguyễn Thành Đạt <br> Đinh Văn Ty <br> Phan Lê Thanh Hoàng <br> Hoàng Minh Hải <br> Từ Phúc Nguyên <br> Nguyễn Văn Toàn |
| **Database path** | RDS SQL Server|
| **W2 Evidence commit** | [none](https://github.com/org/repo/commit/abc123) |
| **Tuần** | W3 — 20-24/04/2026  |



---

## Section 1 — W2 Recap

**Diagram W2:** ![image](https://hackmd.io/_uploads/HJKSVJuaWx.png)

<!-- EDIT: Paste link/ảnh diagram W2 -->

**1 feedback cụ thể từ W2 và cách W3 fix:**

| W2 Feedback | Cách W3 address |
|-------------|-----------------|
| RDS chưa encryption  | RDS encryption at rest |


---

## Section 2 — Data Access Pattern Log

### Part A - 3 Access Patterns

| # | Access Pattern | Tần suất ước lượng |
|---|---|---|
| 1 | Lấy toàn bộ đơn hàng của một user, sắp xếp theo ngày giảm dần | ~50 lần/phút lúc cao điểm |
| 2 | Tra cứu product theo ID | ~40 lần/phút lúc cao điểm |
| 3 | Tạo một order mới cùng nhiều order item theo kiểu atomic | ~10 lần/phút lúc cao điểm |

---

### Part B - Engine + Paradigm + Mechanism cho từng pattern

| # | Pattern | Paradigm | Engine | Mechanism | Vì sao hiệu quả |
|---|---|---|---|---|---|
| 1 | Lấy toàn bộ đơn hàng của một user, sắp xếp theo ngày giảm dần | Relational | RDS SQL Server | Non-clustered composite index `idx_order_user_date` trên bảng `[ORDER](UserID, OrderDateTime DESC)` kết hợp với truy vấn `WHERE UserID = @UserID ORDER BY OrderDateTime DESC` | SQL Server có thể dùng `Index Seek` theo `UserID`, sau đó đọc dữ liệu theo thứ tự `OrderDateTime DESC` ngay trên index. Cách này giảm chi phí quét bảng và tránh bước sort bổ sung, phù hợp với màn hình lịch sử đơn hàng. |
| 2 | Tra cứu product theo ID | Relational | RDS SQL Server | Primary key trên `PRODUCT(ProductID)` | Đây là dạng point lookup theo khóa chính. SQL Server có thể truy xuất trực tiếp đúng record với độ trễ thấp và hiệu năng ổn định, phù hợp với trang chi tiết sản phẩm và các luồng đọc dữ liệu liên quan đến cart. |
| 3 | Tạo một order mới cùng nhiều order item theo kiểu atomic | Relational | RDS SQL Server | `BEGIN TRANSACTION / COMMIT`, `OUTPUT INSERTED.OrderID`, kết hợp `WITH (UPDLOCK, ROWLOCK)` khi đọc tồn kho từ `PRODUCT_SIZE` trước khi cập nhật số lượng | Luồng checkout phải tạo order, tạo nhiều order item và giảm tồn kho trong cùng một transaction ACID. Nếu bất kỳ bước nào lỗi thì toàn bộ transaction sẽ rollback. Cơ chế locking giúp tránh race condition và giảm nguy cơ overselling. |

### Phần C Wrong-Paradigm Test

**Pattern được chọn để test: Pattern #3 - Atomic checkout transaction**

Nếu Pattern #3 được triển khai trên **key-value engine như DynamoDB**, hệ thống sẽ phải dùng `TransactWriteItems` để insert order, insert order item và update tồn kho. Dù DynamoDB có hỗ trợ transactional write, nó vẫn kém phù hợp hơn SQL Server trong use case này vì một số lý do.

Thứ nhất, checkout trong hệ thống này không phải chỉ là một thao tác ghi đơn giản. Nó liên quan tới nhiều record có quan hệ với nhau giữa `ORDER`, `ORDER_ITEM` và `PRODUCT_SIZE`, đồng thời còn phải kiểm tra tồn kho trước khi xác nhận thanh toán. Trong SQL Server, phần logic này phù hợp tự nhiên với relational transaction và ACID guarantee mạnh.

Thứ hai, SQL Server hỗ trợ các chiến lược locking như `UPDLOCK` và `ROWLOCK`, cho phép ứng dụng khóa đúng dòng dữ liệu tồn kho trong lúc kiểm tra và cập nhật số lượng. Điều này đặc biệt quan trọng khi nhiều user cùng mua một size sản phẩm trong cùng thời điểm. DynamoDB không có mô hình row-level locking tương đương, nên việc chống overselling sẽ phức tạp hơn và thường phải dựa vào conditional write cùng logic bổ sung ở tầng application.

Thứ ba, dữ liệu nghiệp vụ của hệ thống e-commerce này có quan hệ rõ ràng: một user có nhiều order, một order có nhiều order item, và mỗi order item tham chiếu tới một product size cụ thể. SQL Server xử lý kiểu dữ liệu quan hệ này tốt hơn nhờ primary key, foreign key, transaction integrity và các indexed query.

Vì vậy, **relational database dùng SQL Server** là lựa chọn phù hợp hơn cho pattern checkout của hệ thống này vì nó cung cấp consistency mạnh hơn, xử lý transaction đơn giản hơn và hỗ trợ tốt hơn cho dữ liệu nghiệp vụ có quan hệ.

---

## Section 3 — Deployment Evidence



### 3.1 Database — Chung (mọi engine)

#### Private Subnet

- [x] Database instance deployed trong **private subnet**, không có public IP

![2 subnet](https://hackmd.io/_uploads/SkkXrB_6Zx.jpg)
![subnet1](https://hackmd.io/_uploads/H1REHB_TWx.jpg)
![subnet2](https://hackmd.io/_uploads/ByPHHr_abx.jpg)



**Notes:** Instance `database-ecommerce` nằm trong subnet group: db_subnet . Route table của subnet không có route `0.0.0.0/0 → igw-*`, chỉ có local route. Vì để bảo mật nên chỉ để db ở tầng private subnet, chỉ cho phép truy cập từ application

---

#### Encryption at Rest

- [x] Encryption at rest **enabled**

![encrypt at rest](https://hackmd.io/_uploads/SJ2sHBOaZl.jpg)



**Notes:** Storage encryption bật với AWS-managed key `aws/rds`. Chọn AWS-managed thay vì customer CMK vì chưa có compliance mandate về key rotation manual và muốn AWS tự rotate hàng năm.
<!-- EDIT: Đổi nếu dùng customer CMK hoặc engine khác -->

---

#### High Availability Plan

- [x] **Multi-AZ enabled** (managed engine) <!-- EDIT: hoặc đổi thành replica plan / SPOF acknowledged nếu self-hosted -->

![Multi-AZ](https://hackmd.io/_uploads/S19_WrOabe.jpg)


<!-- EDIT: Chụp RDS console → tab Configuration → Multi-AZ: Yes -->

**Notes:** Multi-AZ enabled — synchronous standby ở `us-west-2a`, primary ở `us-west-2a`. Vì lý do để cải thiện High Avaiability, khi AC us-west-2a sập thì chúng ta vẫn còn dự phòng ở us-west-2b
<!-- EDIT: Đổi AZ thật -->

---

#### Record Written & Read

- [x] Ít nhất **1 record được write và read** qua application hoặc CLI

![image](https://hackmd.io/_uploads/Hy9k0zOT-e.png)

**Notes:**  
Insert 1 row vào bảng `PRODUCT` thông qua `sqlcmd` CLI từ EC2, sau đó thực hiện truy vấn `SELECT` theo khóa chính (`ProductID`) để xác minh dữ liệu đã được ghi thành công và trả về đúng giá trị. Đây là một ví dụ của **indexed lookup** trong relational database.

```sql
INSERT INTO dbo.PRODUCT (ProductID, Name, Brand, Description, ImageURL)
VALUES (100, 'Demo Product', 'Demo Brand', 'Inserted from EC2 using sqlcmd', 'https://example.com/demo-product.png');

SELECT ProductID, Name, Brand, Description, ImageURL
FROM dbo.PRODUCT
WHERE ProductID = 100;
```

### 3.2 Database — Paradigm Specific

#### [Relational] Schema — ≥2 Related Tables với Foreign Key

- [x] Schema có **ít nhất 2 bảng có quan hệ** với foreign key constraint

![image](https://hackmd.io/_uploads/S1_2vBu6Zl.png)


**Notes:** Bảng `order` (`order_id` PK) và `order_item` (`OrderitemId` PK, `order_id` FK). Để tránh dư thừa dữ liệu và đảm bảo tính toàn vẹn.

---

#### [Relational] Indexed Lookup

- [x] Có **index hỗ trợ WHERE / JOIN** cho access pattern phổ biến


| Table | Index | Type | Columns | PK | Unique |
|---|---|---|---|:---:|:---:|
| ORDER | idx_order_user_date | NONCLUSTERED | UserID, OrderDateTime | — | — |
| ORDER | PK__ORDER__C3905BAF… | CLUSTERED | OrderID | ✓ | ✓ |
| ORDER_ITEM | idx_order_item_order_id | NONCLUSTERED | OrderID | — | — |
| ORDER_ITEM | PK__ORDER_IT__57ED06A1… | CLUSTERED | OrderItemID | ✓ | ✓ |
| PRODUCT | PK__PRODUCT__B40CC6ED… | CLUSTERED | ProductID | ✓ | ✓ |
| PRODUCT_SIZE | PK__PRODUCT___9DADF571… | CLUSTERED | ProductSizeID | ✓ | ✓ |
| USER | PK__USER__1788CCAC… | CLUSTERED | UserID | ✓ | ✓ |

**Notes:**  
Dùng index `idx_order_user_date` để filter theo `UserID` và sort `OrderDateTime DESC`, và `idx_order_item_order_id` để tối ưu JOIN theo `OrderID`.
```sql
CREATE NONCLUSTERED INDEX idx_order_user_date
    ON [ORDER](UserID ASC, OrderDateTime DESC);
    
CREATE NONCLUSTERED INDEX idx_order_item_order_id
    ON [ORDER_ITEM](OrderID ASC);
```
---

#### [Relational] Automated Backups — 7+ ngày retention

- [x] Automated backups configured với **retention ≥7 ngày**

![Backup config](https://hackmd.io/_uploads/Hyi6Mrda-e.jpg)

<!-- EDIT: Chụp RDS console → Maintenance & backups → Automated backups: 7 days -->

**Notes:** Automated backup retention 7 ngày. Backup window: 18:00-20:00 UTC (low-traffic) vì để giảm thiểu ảnh hưởng hiệu năng


### 3.3 VPC & Networking

- [x] VPC diagram show **3 tiers có label**: Public / Private Application / Private Database

![image](https://hackmd.io/_uploads/BJTd3rdTZg.png)



---

- [x] **S3 Gateway Endpoint** provision và labeled trên diagram (với route table entry)

![image](https://hackmd.io/_uploads/BytM1UDpZx.png)


**Notes:** VPC Gateway Endpoint `vpce-0945ef943cecc4994` cho S3. Route table `rtb-051e518c8e9856fb5` có entry `pl-68a54001 (com.amazonaws.us-west-2.s3) → vpce-0945ef943cecc4994`. S3 traffic từ private subnet sẽ đi qua VPC Gateway EndPoint mà không đi qua NAT Gateway, giúp tiết kiệm cost.
<!-- EDIT: Đổi vpce ID, rtb ID, prefix list ID thật -->

---

- [x] Database Security Group inbound rule source = **App-tier Security Group ID** (không phải CIDR)

![image](https://hackmd.io/_uploads/Hk-08vvaZl.png)



**Notes:** DB SG `sg-05fcb3d245482d04a` (db tier SG ID) inbound rule: Port 1433 (SQLServer), Source = `sg-09ea139c58a3e1c16` (app tier SG ID).
<!-- EDIT: Đổi SG IDs thật -->

---


**Security Group over NACL:** Ưu tiên sử dụng Security Group vì tính chất Stateful, tự động cho phép luồng dữ liệu phản hồi mà không cần mở thủ công các cổng ephemeral ports như NACL (Stateless), giúp giảm thiểu sai sót cấu hình. Quan trọng hơn, SG cho phép quản lý bảo mật theo Security Group ID (như việc chỉ cho phép sg-app truy cập DB), giúp thực thi nguyên tắc Least Privilege linh hoạt theo thực thể thay vì bị bó hẹp trong dải IP cố định của Subnet như NACL. Với cơ chế mặc định 'Cấm tất cả', SG là chốt chặn an toàn nhất để vượt qua các bài kiểm tra truy cập trái phép (Negative Security Test).

---

## Section 4 — Working Query Evidence


### Operation 1 — JOIN Query (Relational)

**Mô tả:** Lấy danh sách tất cả các order items của người dùng có UserID = 1, bằng cách JOIN giữa hai bảng dbo.[ORDER] và dbo.ORDER_ITEM thông qua khóa OrderID, và sắp xếp kết quả theo thời gian đặt hàng giảm dần.

```sql
SELECT
    o.OrderID,
    o.OrderDateTime,
    o.TotalPrice,
    o.OrderStatus,
    oi.OrderItemID,
    oi.Quantity,
    oi.ProductSizeID
FROM dbo.[ORDER] o
JOIN dbo.ORDER_ITEM oi ON o.OrderID = oi.OrderID
WHERE o.UserID = 1
ORDER BY o.OrderDateTime DESC, oi.OrderItemID ASC;
```
![image](https://hackmd.io/_uploads/r14xKBdTbg.png)


## Section 5 — Bedrock
### 5.1 Create knowledge file for Bedrock 
 Tạo **3 documents** làm dữ liệu đầu vào cho Knowledge Base:
- `product_faq.txt`
- `return_refund_policy.txt`
- `shipping_policy.txt`
![image](https://hackmd.io/_uploads/S1684wva-e.png)


### 5.2 Create S3 bucket for knowledge base

- Truy cập AWS Console → S3 → Create bucket  
- Đặt tên bucket: `group3-kbai` (đảm bảo unique)  
- Chọn region: ap-southeast-1   
- Giữ cấu hình mặc định  
![Ảnh chụp màn hình 2026-04-23 154527](https://hackmd.io/_uploads/Sk3XXvDpbx.png)
---

- Cấu hình bảo mật và encryption 
![Ảnh chụp màn hình 2026-04-23 154606](https://hackmd.io/_uploads/SJn7QvvTWe.png)
**Notes:**  
- Block Public Access: ON  
- Object Ownership: ACLs disabled  
- Default encryption: SSE-S3 (mặc định)

![Ảnh chụp màn hình 2026-04-23 154624](https://hackmd.io/_uploads/HkTNQPD6Zg.png)

**Files uploaded:**
- `product_faq.txt`
- `return_refund_policy.txt`
- `shipping_policy.txt`
![image](https://hackmd.io/_uploads/SkKlD5v6Wl.png)

- [x] Bucket sẵn sàng làm data source cho Knowledge Base

---

### 5.3 Create bedrock knowledge base  

#### 5.3.1 Tạo Knowledge Base  

- Vào **Amazon Bedrock**  
- Chọn **Knowledge Bases** → **Create knowledge base**  

Thông tin cơ bản:
- Name: `group3-aikb`  
- Description: `Dữ liệu được phân tách theo từng nhóm nội dung, hỗ trợ truy vấn chính xác và phục vụ pipeline retrieve–generate`
  
  
![Ảnh chụp màn hình 2026-04-23 154710](https://hackmd.io/_uploads/S1DZEPwp-e.png)

#### 5.3.2 Cấu hình Data Source (S3)  

- **Source name:** `S3Source`  
- **S3 URI:** chọn bucket `group3-ai` (Browse S3)  

![Ảnh chụp màn hình 2026-04-23 154732](https://hackmd.io/_uploads/HkPbVPvTWx.png)
![Ảnh chụp màn hình 2026-04-23 154750](https://hackmd.io/_uploads/SJvWNDvTZx.png)
---
#### 5.3.3 Chọn Embedding Model  
- Sử dụng **Cohere Embedding Model** (multilingual hoặc tương đương)  
---
#### 5.3.4 Cấu hình Vector Store  
- Chọn:  
  - **Quick create a new vector store**  
  - Loại: **S3 vector store**  

![Ảnh chụp màn hình 2026-04-23 154808](https://hackmd.io/_uploads/SJv-NvPpWg.png)
---
#### 5.3.5 Tạo Knowledge Base  

- Kiểm tra lại toàn bộ cấu hình  
- Nhấn **Create knowledge base**  

---

#### 5.3.6 Đồng bộ dữ liệu (bắt buộc)  

- Trong mục **Data source**  
- Nhấn **Sync**  
- Đợi trạng thái: `Complete`  
![Ảnh chụp màn hình 2026-04-23 154850](https://hackmd.io/_uploads/HyK_2HOp-x.png)
![Ảnh chụp màn hình 2026-04-23 154901](https://hackmd.io/_uploads/Sk5uhBdT-x.png)

---

#### 5.3.7 Lưu thông tin quan trọng  

Sau khi hoàn tất, lưu lại:  

- **Knowledge Base ID** (phần Overview)  
- **Data Source ID** (bảng Data source phía dưới)  

---

**Ghi chú**  

- Nếu chưa **Sync** → hệ thống chưa index dữ liệu → truy vấn sẽ không có kết quả  
- Embedding model ảnh hưởng đến chất lượng tìm kiếm  
- S3 vector store phù hợp cho demo (triển khai nhanh, đơn giản)  
---
### 5.4 Create lambda funtion
Triển khai Lambda Function (Chatbot Orchestrator)

---
![Ảnh chụp màn hình 2026-04-24 085533](https://hackmd.io/_uploads/BkQhTSO6Zl.png)

**1. Tạo Lambda Function**

- Truy cập **AWS Lambda**
- Chọn **Create function**

**Cấu hình:**
- Function name: `Chatbot_Orchestrator`
- Runtime: `Python 3.12`

→ Nhấn **Create function**


**2. Cập nhật Code**

- Xóa toàn bộ code mặc định
- Dán đoạn code vào:
```
import boto3
import json
import logging


# Cấu hình logging chuẩn của AWS
logger = logging.getLogger()
logger.setLevel(logging.INFO)


REGION = 'ap-southeast-1'
bedrock_agent = boto3.client('bedrock-agent', region_name=REGION)
bedrock_runtime = boto3.client('bedrock-agent-runtime', region_name=REGION)


def lambda_handler(event, context):
   
    kb_id = 'IZAUMZEQYU'
    ds_id = 'STHKG4IJSR'
   
    # 1. Xử lý S3 Sync (Trigger từ S3)
    if 'Records' in event and event['Records'][0].get('eventSource') == 'aws:s3':
        bucket = event['Records'][0]['s3']['bucket']['name']
        file_name = event['Records'][0]['s3']['object']['key']
        logger.info(f"Phát hiện file mới: {file_name} tại bucket: {bucket}. Đang bắt đầu Sync...")
       
        try:
            response = bedrock_agent.start_ingestion_job(knowledgeBaseId=kb_id, dataSourceId=ds_id)
            logger.info(f"Sync đã kích hoạt thành công. Job ID: {response['ingestionJob']['ingestionJobId']}")
            return {'statusCode': 200, 'body': 'Auto-Sync Started'}
        except Exception as e:
            logger.error(f"Lỗi khi Sync: {str(e)}")
            return {'statusCode': 500, 'body': str(e)}


    # 2. Xử lý Chat (Gọi từ Backend/Test)
    else:
        body = event.get('body', event)
        if isinstance(body, str): body = json.loads(body)
        user_query = body.get('question')
       
        if not user_query:
            logger.warning("Request không có câu hỏi (question)")
            return {'statusCode': 400, 'body': 'Thiếu câu hỏi trong JSON'}


        logger.info(f"Đang hỏi Bedrock câu: {user_query}")


        try:
            model_arn = f'arn:aws:bedrock:{REGION}::foundation-model/anthropic.claude-3-haiku-20240307-v1:0'


            response = bedrock_runtime.retrieve_and_generate(
                input={'text': user_query},
                retrieveAndGenerateConfiguration={
                    'type': 'KNOWLEDGE_BASE',
                    'knowledgeBaseConfiguration': {
                        'knowledgeBaseId': kb_id,
                        'modelArn': model_arn
                    }
                }
            )
           
            logger.info("Bedrock đã phản hồi thành công.")
            return {
                'statusCode': 200,
                'headers': {'Content-Type': 'application/json'},
                'body': json.dumps({
                    'answer': response['output']['text'],
                    'evidence': 'Bedrock Knowledge Base'
                }, ensure_ascii=False)
            }
        except Exception as e:
            logger.error(f"Lỗi khi gọi Bedrock: {str(e)}")
            return {'statusCode': 500, 'body': f"Lỗi hệ thống: {str(e)}"}
```

**3. Cấu hình ID quan trọng**

Trong code, thay thế:

- `YOUR_KB_ID` → bằng **Knowledge Base ID**
- `YOUR_DATA_SOURCE_ID` → bằng **Data Source ID**

*(2 giá trị này lấy từ bước tạo Knowledge Base)*

**4. Deploy**

- Nhấn **Deploy** để lưu và kích hoạt function


**Ghi chú**

- Đảm bảo Lambda đã được gán:
  - `Chatbot_Production_Policy` (IAM)
- Nếu sai ID:
  - Query sẽ không trả về dữ liệu
- Nếu chưa Sync Knowledge Base:
  - RAG sẽ không hoạt động

---
![Ảnh chụp màn hình 2026-04-23 154936](https://hackmd.io/_uploads/BkCuEPvpbl.png)


---

### 5.5 Attache policy for lambda
#### Hướng dẫn tạo IAM Policy cho Chatbot (Bedrock + S3 + Lambda)

Thực hiện tại: **IAM > Policies > Create Policy (Visual Editor)**

---

#### Bước 1: Quyền cho Bedrock Knowledge Base

- **Service:** Bedrock  
- **Actions:**
  - Retrieve  
  - RetrieveAndGenerate  
  - StartIngestionJob  

- **Resources:** Specific  
  - knowledge-base → Add ARNs:
    - Region: `ap-southeast-1`
    - Account: `032265101228`
    - Knowledge base ID: `IZAUMZEQYU`

---

#### Bước 2: Quyền gọi Model (InvokeModel)

- **Add more permissions**
- **Service:** Bedrock  
- **Actions:**
  - InvokeModel  

- **Resources:** Specific  
  - foundation-model → Add ARNs:
    - Region: `ap-southeast-1` *(hoặc để trống)*
    - Model ID: `anthropic.claude-3-haiku-20240307-v1:0`

---

#### Bước 3: Quyền S3 (Lưu trữ)

- **Add more permissions**
- **Service:** S3  
- **Actions:**
  - GetObject  
  - ListBucket  

- **Resources:** Specific  

  **Bucket:**
  - Bucket name: `toan-chatbot-data-2026`

  **Object:**
  - Bucket name: `toan-chatbot-data-2026`
  - Object name: `*`

---

#### Bước 4: Quyền CloudWatch Logs

- **Add more permissions**
- **Service:** CloudWatch Logs  
- **Actions:**
  - CreateLogGroup  
  - CreateLogStream  
  - PutLogEvents  

- **Resources:** Specific  

  **Log Group:**
  - Region: `ap-southeast-1`
  - Account: `032265101228`
  - Log group name: `/aws/lambda/Chatbot_Orchestrator`

  **Log Stream:**
  - Region: `ap-southeast-1`
  - Account: `032265101228`
  - Log group name: `/aws/lambda/Chatbot_Orchestrator`
  - Log stream name: `*`

---

#### Bước 5: Đặt tên Policy

- **Policy name:** `Chatbot_Production_Policy`  
- **Description:**  
![ai1](https://hackmd.io/_uploads/SJSEHDvpbx.png)
![ai2](https://hackmd.io/_uploads/BJS4SwDaZg.png)
```
# IAM Policy (JSON)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VisualEditor0",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "logs:CreateLogStream",
        "bedrock:InvokeModel",
        "s3:ListBucket",
        "logs:CreateLogGroup",
        "logs:PutLogEvents",
        "bedrock:Retrieve"
      ],
      "Resource": [
        "arn:aws:s3:::group3-kb",
        "arn:aws:s3:::group3-kb/*",
        "arn:aws:logs:us-west-2:664945433260:log-group:/aws/lambda/group3-kb",
        "arn:aws:logs:us-west-2:664945433260:log-group:/aws/lambda/group3-kb:log-stream:*",
        "arn:aws:bedrock:us-west-2:664945433260:knowledge-base/ORLPNINRTA",
        "arn:aws:bedrock:us-west-2::foundation-model/anthropic.claude-3-haiku-20240307-v1:0"
      ]
    },
    {
      "Sid": "VisualEditor1",
      "Effect": "Allow",
      "Action": "bedrock:RetrieveAndGenerate",
      "Resource": "*"
    }
  ]
}
```



### 5.6 Config Sync automation when upload S3
![Ảnh chụp màn hình 2026-04-23 164513](https://hackmd.io/_uploads/B1LF5DPabg.png)
![Ảnh chụp màn hình 2026-04-23 164523](https://hackmd.io/_uploads/H1lUK9wPabe.png)
![Ảnh chụp màn hình 2026-04-23 164532](https://hackmd.io/_uploads/SyItqvv6Zg.png)
![Ảnh chụp màn hình 2026-04-23 164543](https://hackmd.io/_uploads/r1wF9DwTZl.png)
![Ảnh chụp màn hình 2026-04-23 164553](https://hackmd.io/_uploads/HJvtqDv6bx.png)
![Ảnh chụp màn hình 2026-04-23 164609](https://hackmd.io/_uploads/BJwF9wvT-x.png)





### 5.7 Show evidence
![Ảnh chụp màn hình 2026-04-23 164742](https://hackmd.io/_uploads/S1ngiPPpWx.png)
![Ảnh chụp màn hình 2026-04-23 164753](https://hackmd.io/_uploads/By2ejPPpbl.png)
![Ảnh chụp màn hình 2026-04-23 164800](https://hackmd.io/_uploads/By3esDwa-x.png)


---

## Section 6 — VPC + Networking Evidence

### Route Table — S3 Gateway Endpoint

Config trên console:
![image](https://hackmd.io/_uploads/r14ZV8PTZl.png)

**Entry trong route table:**

| Destination | Target | Status |
|-------------|--------|--------|
| 10.0.0.0/16 | local | Active |
| pl-68a54001 (com.amazonaws.us-west-2.s3) | vpce-0945ef943cecc4994 | Active |
| 0.0.0.0/0 | nat-1a73daa07eec40863 | Active |


<!-- EDIT: Đổi prefix list ID, vpce ID, NAT GW ID thật -->

**Notes:** Entry `pl-68a54001 → vpce-0945ef943cecc4994` là S3 Gateway Endpoint. Traffic từ Lambda/app tier tới S3 đi qua VPC endpoint, không qua NAT Gateway — không tốn per-GB NAT processing fee (~$0.045/GB), và ở lại trong AWS backbone (không ra internet).
<!-- EDIT: Điều chỉnh reasoning thật -->

---

### DB Security Group Inbound Rules

Config trên console:
![image](https://hackmd.io/_uploads/B1bEvvDp-g.png)

**Inbound rules của DB SG `sg-db-xbrain`:**

| Type | Protocol | Port | Source | Description |
|------|----------|------|--------|-------------|
| SQLServer | TCP | 1433 | sg-09ea139c58a3e1c16 / app  | App tier access only |

<!-- EDIT: Đổi SG ID, port (3306 nếu MySQL), description thật -->

---

## Section 7 — Negative Security Test

**Kịch bản:** Thử connect trực tiếp tới

---
 RDS endpoint từ máy local (ngoài VPC) — không qua bastion, không có VPN.

```bash
# EDIT: Đổi hostname RDS endpoint thật
Test-NetConnection database-ecommerce.clesw26c47c1.us-west-2.rds.amazonaws.com -Port 1433

# Expected output (kết nối bị từ chối / timeout):
```
![image](https://hackmd.io/_uploads/Bk30pPwT-g.png)



**Notes:** DB Security Group `sg-05fcb3d245482d04a` chỉ có 1 inbound rule: port 1433 từ `sg-09ea139c58a3e1c16` (app-tier SG). Không có rule cho `0.0.0.0/0` hoặc bất kỳ source ngoài app-tier SG. Kết nối từ máy local bị drop silently ở SG level — "Connection timed out" (không phải "Connection refused") vì SG drop packet, không reset connection.
<!-- EDIT: Đổi SG IDs thật, có thể dùng kịch bản denied khác như unauthorized IAM action -->

---

## Section 8 — Bonus (Tùy Chọn)

> Chỉ điền nếu đã hoàn thành tất cả must-haves + Evidence Pack. Trainer chấm bonus sau khi verify sections 1-7.

<!-- EDIT: Chọn 1 scenario. Xóa comment và điền nội dung. -->

### Bonus C — Partial CloudFormation Template (+0.25)

**Scenario:** Viết partial CloudFormation template cho việc tạo VPC, Subnet, Route,... .

**Pre-state:** Không có IaC — resource được tạo thủ công qua console.

**Template (`network.yaml`):**
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Network Layer

# Parameters: biến đầu vào, có thể truyền khi deploy
# Nếu không truyền thì dùng Default
Parameters:
  VpcCIDR:
    Type: String
    Default: 10.1.0.0/16

Resources:

  # ===== VPC =====
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCIDR          # !Ref lấy giá trị từ Parameter VpcCIDR ở trên
      EnableDnsHostnames: true          # EC2 có hostname dạng ip-10-1-x-x.region.compute.internal
      EnableDnsSupport: true            # Cho phép DNS resolution trong VPC
      Tags:
        - Key: Name
          Value: HAI-VPC

  # IGW tạo ra độc lập, chưa tự gắn vào VPC
  IGW:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: HAI-IGW

  # Resource riêng để gắn IGW vào VPC
  # !Ref VPC → tự động lấy ID của resource VPC vừa tạo ở trên
  AttachIGW:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref IGW

  # ===== Subnets (2 AZs) =====
  # !GetAZs "" → trả về list AZ của region hiện tại, ví dụ ["us-west-2a", "us-west-2b", ...]
  # !Select [0, ...] → lấy phần tử index 0 → AZ đầu tiên (us-west-2a)
  # !Select [1, ...] → lấy phần tử index 1 → AZ thứ hai (us-west-2b)
  # MapPublicIpOnLaunch: true → EC2 launch vào subnet này tự nhận Public IP

  PublicSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.1.1.0/24
      AvailabilityZone: !Select [0, !GetAZs ""]
      MapPublicIpOnLaunch: true         # Chỉ Public Subnet mới bật cái này
      Tags:
        - Key: Name
          Value: HAI-PublicSubnet-A

  PublicSubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.1.2.0/24
      AvailabilityZone: !Select [1, !GetAZs ""]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: HAI-PublicSubnet-B

  # App và Data Subnet không có MapPublicIpOnLaunch → mặc định false → chỉ có Private IP
  AppSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.1.3.0/24
      AvailabilityZone: !Select [0, !GetAZs ""]
      Tags:
        - Key: Name
          Value: HAI-AppSubnet-A

  AppSubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.1.4.0/24
      AvailabilityZone: !Select [1, !GetAZs ""]
      Tags:
        - Key: Name
          Value: HAI-AppSubnet-B

  DataSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.1.5.0/24
      AvailabilityZone: !Select [0, !GetAZs ""]
      Tags:
        - Key: Name
          Value: HAI-DataSubnet-A

  DataSubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.1.6.0/24
      AvailabilityZone: !Select [1, !GetAZs ""]
      Tags:
        - Key: Name
          Value: HAI-DataSubnet-B

  # ===== NAT Gateway =====
  # EIP: Elastic IP tĩnh gắn vào NAT, IP này không đổi dù restart
  EIP:
    Type: AWS::EC2::EIP
    Properties:
      Tags:
        - Key: Name
          Value: HAI-EIP

  # NAT phải đặt trong Public Subnet vì nó cần đường ra internet qua IGW
  # !GetAtt khác !Ref: dùng để lấy thuộc tính con của resource
  # EIP.AllocationId → ID cấp phát của EIP (NAT cần cái này, không cần IP trực tiếp)
  NAT:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt EIP.AllocationId
      SubnetId: !Ref PublicSubnetA
      Tags:
        - Key: Name
          Value: HAI-NAT

  # ===== Route Tables =====
  PublicRT:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: HAI-PublicRT

  # DependsOn: bắt buộc IGW phải gắn vào VPC xong rồi mới tạo route
  # Nếu không có DependsOn, route có thể được tạo trước khi IGW sẵn sàng → lỗi
  # DestinationCidrBlock: 0.0.0.0/0 → route mặc định, mọi traffic không khớp rule nào đi theo đây
  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachIGW
    Properties:
      RouteTableId: !Ref PublicRT
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref IGW              # Public → ra internet trực tiếp qua IGW

  # App Subnet dùng route table này, có NAT để ra internet
  PrivateRT:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: HAI-AppRT

  PrivateRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRT
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NAT           # App → ra internet gián tiếp qua NAT (ẩn IP thật)

  # DB dùng route table riêng, KHÔNG có route 0.0.0.0/0
  # → DB không có đường ra internet, cô lập hoàn toàn
  DataRT:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: HAI-DataRT

  # ===== Gắn Subnet vào Route Table =====
  PublicAssocA:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetA
      RouteTableId: !Ref PublicRT

  PublicAssocB:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetB
      RouteTableId: !Ref PublicRT

  PrivateAssoc1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref AppSubnetA
      RouteTableId: !Ref PrivateRT

  PrivateAssoc2:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref AppSubnetB
      RouteTableId: !Ref PrivateRT

  DataAssoc1:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref DataSubnetA
      RouteTableId: !Ref DataRT

  DataAssoc2:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref DataSubnetB
      RouteTableId: !Ref DataRT

  # ===== VPC Endpoint (S3 Gateway) =====
  # Thay vì traffic đến S3 phải ra internet, VPCE tạo đường đi nội bộ trong AWS
  # → Không tốn NAT bandwidth, bảo mật hơn, không tính phí data transfer ra internet
  # !Sub thay thế biến trong string: ${AWS::Region} là Pseudo Parameter, AWS tự điền region
  # → com.amazonaws.us-west-2.s3
  # RouteTableIds: inject route đặc biệt vào các route table này
  # → chỉ App cần, DB không cần vì backup đi qua App
  S3Endpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VPC
      ServiceName: !Sub com.amazonaws.${AWS::Region}.s3
      RouteTableIds:
        - !Ref PrivateRT               # App → S3 qua VPCE

# Outputs: export giá trị ra ngoài để stack khác dùng !ImportValue
# Nếu không có Export thì stack khác không thể tham chiếu tới
Outputs:
  VpcId:
    Value: !Ref VPC
    Export:
      Name: VpcId

  PublicSubnetA:
    Value: !Ref PublicSubnetA
    Export:
      Name: PublicSubnetA

  PublicSubnetB:
    Value: !Ref PublicSubnetB
    Export:
      Name: PublicSubnetB

  AppSubnetA:
    Value: !Ref AppSubnetA
    Export:
      Name: AppSubnetA

  AppSubnetB:
    Value: !Ref AppSubnetB
    Export:
      Name: AppSubnetB

  DataSubnetA:
    Value: !Ref DataSubnetA
    Export:
      Name: DataSubnetA

  DataSubnetB:
    Value: !Ref DataSubnetB
    Export:
      Name: DataSubnetB


```

**Validate command:**
```bash
aws cloudformation create-stack --stack-name network 
--template-body file://network.yaml
aws cloudformation wait stack-create-complete --stack-name network
```

**Expected output:**
```json
{
    "StackId": "arn:aws:cloudformation:us-west-2:664945433260:stack/network/9e573220-3eea-11f1-bb6e-06e166181a0d",
    "OperationId": "9e586aa0-3eea-11f1-bb6e-06e166181a0d"
}
```

**Template (`security.yaml`):**

**Scenario:** Viết partial CloudFormation template cho việc tạo các SGs, NACLs.

**Pre-state:** Không có IaC — resource được tạo thủ công qua console.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Security Layer

Resources:

  # ===== Security Groups =====
  # Security Group là stateful: chỉ cần allow inbound, outbound response tự động được phép
  # Outbound mặc định allow all nếu không khai báo SecurityGroupEgress
  # !ImportValue: lấy giá trị đã Export từ stack khác (ở đây là stack network)

  ExternalALBSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: HAI-sg-external-alb    # Tên hiển thị trên Console, không có thì AWS tự đặt tên random
      GroupDescription: External ALB - allow HTTP and HTTPS from internet
      VpcId: !ImportValue VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp               # tcp=6, udp=17, icmp=1, -1=all
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0            # Cho phép từ mọi IP trên internet
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: HAI-sg-external-alb

  WebSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: HAI-sg-web
      GroupDescription: Web tier - allow HTTP from External ALB only
      VpcId: !ImportValue VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          # SourceSecurityGroupId thay vì CidrIp:
          # → không dùng IP cứng mà dùng SG làm source
          # → "chỉ cho phép traffic từ resource nào đang attach ExternalALBSG"
          # → linh hoạt hơn vì IP của ALB có thể thay đổi
          SourceSecurityGroupId: !Ref ExternalALBSG
      Tags:
        - Key: Name
          Value: HAI-sg-web

  InternalALBSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: HAI-sg-internal-alb
      GroupDescription: Internal ALB - allow HTTP from Web tier only
      VpcId: !ImportValue VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !Ref WebSG   # Chỉ Web mới gọi vào Internal ALB
      Tags:
        - Key: Name
          Value: HAI-sg-internal-alb

  AppSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: HAI-sg-app
      GroupDescription: App tier - allow from Internal ALB only
      VpcId: !ImportValue VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 9000
          ToPort: 9000
          SourceSecurityGroupId: !Ref InternalALBSG   # Chỉ Internal ALB mới gọi vào App
      Tags:
        - Key: Name
          Value: HAI-sg-app

  DBSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: HAI-sg-db
      GroupDescription: DB tier - allow MySQL from App tier only
      VpcId: !ImportValue VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3306                # MySQL port
          ToPort: 3306
          SourceSecurityGroupId: !Ref AppSG   # Chỉ App mới gọi vào DB
      Tags:
        - Key: Name
          Value: HAI-sg-db

  EndpointSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: HAI-sg-endpoint
      GroupDescription: VPC Endpoint - allow HTTPS from App tier only
      VpcId: !ImportValue VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          SourceSecurityGroupId: !Ref AppSG   # Chỉ App mới dùng VPC Endpoint
      Tags:
        - Key: Name
          Value: HAI-sg-endpoint

  # ===== NACL cho Public Subnet =====
  # NACL là stateless: phải khai báo cả inbound lẫn outbound
  # Rule được xử lý theo thứ tự RuleNumber từ nhỏ → lớn, khớp rule nào thì áp dụng luôn
  # Protocol: 6=TCP, 17=UDP, -1=tất cả
  # Egress: false=inbound, true=outbound
  NACL:
    Type: AWS::EC2::NetworkAcl
    Properties:
      VpcId: !ImportValue VpcId
      Tags:
        - Key: Name
          Value: HAI-NACL

  InboundHTTP:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref NACL
      RuleNumber: 100
      Protocol: 6
      RuleAction: allow
      Egress: false                     # Inbound
      CidrBlock: 0.0.0.0/0
      PortRange:
        From: 80
        To: 80

  InboundHTTPS:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref NACL
      RuleNumber: 110
      Protocol: 6
      RuleAction: allow
      Egress: false
      CidrBlock: 0.0.0.0/0
      PortRange:
        From: 443
        To: 443

  # Ephemeral ports (1024-65535): NACL stateless nên cần allow dải port này
  # Khi client gọi vào port 80, server trả response về port ngẫu nhiên phía client (trong dải này)
  # Nếu không allow thì response bị chặn dù request vào được
  InboundEphemeral:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref NACL
      RuleNumber: 200
      Protocol: 6
      RuleAction: allow
      Egress: false
      CidrBlock: 0.0.0.0/0
      PortRange:
        From: 1024
        To: 65535

  OutboundHTTP:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref NACL
      RuleNumber: 100
      Protocol: 6
      RuleAction: allow
      Egress: true                      # Outbound
      CidrBlock: 0.0.0.0/0
      PortRange:
        From: 80
        To: 80

  OutboundHTTPS:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref NACL
      RuleNumber: 110
      Protocol: 6
      RuleAction: allow
      Egress: true
      CidrBlock: 0.0.0.0/0
      PortRange:
        From: 443
        To: 443

  # Trả response về cho client qua ephemeral ports
  OutboundEphemeral:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref NACL
      RuleNumber: 200
      Protocol: 6
      RuleAction: allow
      Egress: true
      CidrBlock: 0.0.0.0/0
      PortRange:
        From: 1024
        To: 65535

  # Gắn NACL vào subnet, một subnet chỉ gắn được 1 NACL tại một thời điểm
  NACLAssocPublicA:
    Type: AWS::EC2::SubnetNetworkAclAssociation
    Properties:
      SubnetId: !ImportValue PublicSubnetA
      NetworkAclId: !Ref NACL

  NACLAssocPublicB:
    Type: AWS::EC2::SubnetNetworkAclAssociation
    Properties:
      SubnetId: !ImportValue PublicSubnetB
      NetworkAclId: !Ref NACL

  # ===== NACL cho App Subnet =====
  # Chỉ nhận traffic từ trong VPC (10.1.0.0/16), không nhận từ internet trực tiếp
  AppNACL:
    Type: AWS::EC2::NetworkAcl
    Properties:
      VpcId: !ImportValue VpcId
      Tags:
        - Key: Name
          Value: HAI-AppNACL

  # Chỉ cho Internal ALB (nằm trong VPC) gọi vào port 9000
  AppNACLInboundApp:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref AppNACL
      RuleNumber: 100
      Protocol: 6
      RuleAction: allow
      Egress: false
      CidrBlock: 10.1.0.0/16           # Toàn bộ VPC, không phải 0.0.0.0/0
      PortRange:
        From: 9000
        To: 9000

  # Nhận response trở về từ DB hoặc các service nội bộ (qua ephemeral ports)
  AppNACLInboundEphemeral:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref AppNACL
      RuleNumber: 200
      Protocol: 6
      RuleAction: allow
      Egress: false
      CidrBlock: 10.1.0.0/16
      PortRange:
        From: 1024
        To: 65535

  # Cho App gọi vào DB (port 3306), gọi VPCE, gọi NAT — tất cả đều trong VPC
  AppNACLOutboundVPC:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref AppNACL
      RuleNumber: 100
      Protocol: 6
      RuleAction: allow
      Egress: true
      CidrBlock: 10.1.0.0/16           # Chỉ trong VPC
      PortRange:
        From: 0
        To: 65535

  # Cho App ra internet qua NAT để gọi API ngoài, pull packages
  # Chỉ mở HTTPS (443), không mở HTTP để tránh gọi endpoint không an toàn
  AppNACLOutboundInternet:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref AppNACL
      RuleNumber: 200
      Protocol: 6
      RuleAction: allow
      Egress: true
      CidrBlock: 0.0.0.0/0
      PortRange:
        From: 443
        To: 443

  AppNACLAssocA:
    Type: AWS::EC2::SubnetNetworkAclAssociation
    Properties:
      SubnetId: !ImportValue AppSubnetA
      NetworkAclId: !Ref AppNACL

  AppNACLAssocB:
    Type: AWS::EC2::SubnetNetworkAclAssociation
    Properties:
      SubnetId: !ImportValue AppSubnetB
      NetworkAclId: !Ref AppNACL

  # ===== NACL cho Data Subnet =====
  # Chặt nhất: chỉ nhận từ AppSubnet, không có đường ra internet
  DataNACL:
    Type: AWS::EC2::NetworkAcl
    Properties:
      VpcId: !ImportValue VpcId
      Tags:
        - Key: Name
          Value: HAI-DataNACL

  # Chỉ AppSubnetA (10.1.3.0/24) mới được kết nối MySQL
  DataNACLInboundMySQL:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref DataNACL
      RuleNumber: 100
      Protocol: 6
      RuleAction: allow
      Egress: false
      CidrBlock: 10.1.3.0/24           # Chính xác đến từng subnet, không phải cả VPC
      PortRange:
        From: 3306
        To: 3306

  # Chỉ AppSubnetB (10.1.4.0/24) mới được kết nối MySQL
  DataNACLInboundMySQLB:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref DataNACL
      RuleNumber: 110
      Protocol: 6
      RuleAction: allow
      Egress: false
      CidrBlock: 10.1.4.0/24
      PortRange:
        From: 3306
        To: 3306

  # Nhận ephemeral ports (response từ App sau khi DB gọi ra — ít dùng nhưng cần cho stateless NACL)
  DataNACLInboundEphemeral:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref DataNACL
      RuleNumber: 200
      Protocol: 6
      RuleAction: allow
      Egress: false
      CidrBlock: 10.1.0.0/16
      PortRange:
        From: 1024
        To: 65535

  # Chỉ trả response về AppSubnetA qua ephemeral ports
  # Không có rule 0.0.0.0/0 → DB không thể gửi packet ra ngoài VPC
  DataNACLOutboundToApp:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref DataNACL
      RuleNumber: 100
      Protocol: 6
      RuleAction: allow
      Egress: true
      CidrBlock: 10.1.3.0/24
      PortRange:
        From: 1024
        To: 65535

  # Chỉ trả response về AppSubnetB
  DataNACLOutboundToAppB:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref DataNACL
      RuleNumber: 110
      Protocol: 6
      RuleAction: allow
      Egress: true
      CidrBlock: 10.1.4.0/24
      PortRange:
        From: 1024
        To: 65535

  DataNACLAssocA:
    Type: AWS::EC2::SubnetNetworkAclAssociation
    Properties:
      SubnetId: !ImportValue DataSubnetA
      NetworkAclId: !Ref DataNACL

  DataNACLAssocB:
    Type: AWS::EC2::SubnetNetworkAclAssociation
    Properties:
      SubnetId: !ImportValue DataSubnetB
      NetworkAclId: !Ref DataNACL

# Outputs: export SG ID để stack compute dùng !ImportValue
Outputs:
  ExternalALBSG:
    Value: !Ref ExternalALBSG
    Export:
      Name: ExternalALBSG

  WebSG:
    Value: !Ref WebSG
    Export:
      Name: WebSG

  InternalALBSG:
    Value: !Ref InternalALBSG
    Export:
      Name: InternalALBSG

  AppSG:
    Value: !Ref AppSG
    Export:
      Name: AppSG

  DBSG:
    Value: !Ref DBSG
    Export:
      Name: DBSG

  EndpointSG:
    Value: !Ref EndpointSG
    Export:
      Name: EndpointSG


```
**Validate command:**
```bash
aws cloudformation create-stack --stack-name security 
--template-body file://security.yaml
aws cloudformation wait stack-create-complete --stack-name security
```

**Expected output:**
```json
{
    "StackId": "arn:aws:cloudformation:us-west-2:664945433260:stack/network/9e573220-3eea-11f1-bb6e-06e166181a0d",
    "OperationId": "9e586aa0-3eea-11f1-bb6e-06e166181a0d"
}
```

**Reflection:** Viết CFN template giúp thấy rõ structure của resource hơn so với click console — attribute types phải khai báo trước, key schema phải consistent với attribute definitions. Lần sau sẽ viết template trước rồi mới deploy thay vì làm ngược lại. `validate-template` chỉ check syntax, không check logic (ví dụ: sai region, sai ARN) — cần `create-stack` thật để catch runtime errors.
<!-- EDIT: Viết reflection thật dựa trên trải nghiệm thực tế -->

---

*Evidence Pack hoàn chỉnh. Slides thứ Sáu derive từ file này — copy 8-12 screenshots + captions vào deck và link ngược lại commit markdown này.*
<!-- EDIT: Xóa dòng này sau khi done, hoặc giữ lại như reminder -->
