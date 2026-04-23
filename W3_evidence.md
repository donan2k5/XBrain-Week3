# W3 Evidence Pack — Xương Sống Database

<!-- EDIT: Đổi số nhóm, tên thành viên, engine/paradigm, link W2 commit -->

| Thông tin | Giá trị |
|-----------|---------|
| **Nhóm** | Nhóm X |
| **Thành viên** | Nguyễn Văn A · Trần Thị B · Lê Văn C · Phạm Thị D |
| **Database path** | RDS PostgreSQL / Relational |
| **W2 Evidence commit** | [commit abc123](https://github.com/org/repo/commit/abc123) |
| **Tuần** | W3 — 20-24/4/2026 |

**W2 Trainer Feedback đã được address:**
> Trainer noted IAM policies on EC2 role contained wildcard `Action: "*"`. Fixed in W3: Lambda execution role now scoped to specific actions (`bedrock:RetrieveAndGenerate`, `dynamodb:GetItem`, `s3:GetObject`) and specific resource ARNs. Wildcards removed.
<!-- EDIT: Thay bằng feedback thật từ trainer W2 của nhóm -->

---

## Section 1 — W2 Recap

**Diagram W2:** *(copy hoặc link diagram W2 vào đây)*
<!-- EDIT: Paste link/ảnh diagram W2 -->

**1 feedback cụ thể từ W2 và cách W3 fix:**

| W2 Feedback | Cách W3 address |
|-------------|-----------------|
| IAM role trên EC2 có `Action: "*"` — quá rộng | Lambda execution role W3 scope về `bedrock:RetrieveAndGenerate` + `s3:GetObject` với ARN cụ thể |
<!-- EDIT: Đổi feedback và action thật -->

---

## Section 2 — Data Access Pattern Log

### Part A — 3 Access Patterns Thật Từ App

<!-- EDIT: Đổi tên pattern, frequency cho đúng với app của nhóm -->

| # | Access Pattern | Frequency (ước lượng) |
|---|---------------|----------------------|
| 1 | Get all orders for a user, sorted by `created_at DESC` | ~50 calls/phút lúc peak |
| 2 | Look up product details by `product_id` | ~200 calls/phút |
| 3 | Decrement inventory + insert order trong 1 transaction khi user checkout | ~10 calls/phút |

### Part B — Engine + Paradigm + Mechanism Cho Mỗi Pattern

<!-- EDIT: Đổi paradigm/engine nếu không dùng Relational/RDS -->

| # | Pattern | Paradigm | Engine | Mechanism | Tại sao hiệu quả |
|---|---------|----------|--------|-----------|-----------------|
| 1 | Get orders by user | Relational | RDS PostgreSQL | Index `idx_orders_user_id` trên `orders.user_id` + `ORDER BY created_at DESC` | Index Scan tránh Seq Scan; sort trên indexed column không cần filesort |
| 2 | Lookup product by ID | Relational | RDS PostgreSQL | Primary key lookup trên `products.product_id` (B-tree) | O(1) PK lookup, execution time <1ms |
| 3 | Atomic checkout transaction | Relational | RDS PostgreSQL | `BEGIN` / `COMMIT` ACID transaction qua bảng `orders` + `inventory` | Rollback tự động nếu bất kỳ statement nào fail; `SELECT FOR UPDATE` lock row tránh race condition |

> **Nếu self-hosted trên EC2:** thêm cột "Backup/HA Plan" — ví dụ: *"Daily `pg_dump` cron lúc 02:00 AM, upload lên S3. Read replica trên EC2 instance thứ 2 cùng AZ."*
>
> **Nếu DocumentDB / Neptune:** thêm cột "Monthly Cost Estimate" — ví dụ: *"db.r6g.large ~$220/tháng (on-demand, ap-southeast-1)."*

### Part C — "Wrong-Paradigm" Test

<!-- EDIT: Chọn 1 pattern, giải thích tại sao paradigm khác sẽ fail hoặc expensive -->

**Pattern đã chọn để test: Pattern #3 — Atomic checkout transaction**

Pattern #3 (atomic checkout) trên **key-value engine (DynamoDB)** sẽ cần `TransactWriteItems` qua nhiều items — hết transaction budget 25 items/request nếu order có nhiều line items, và cost per-transactional-request cao hơn standard write. Quan trọng hơn, DynamoDB `TransactWriteItems` không lock rows giữa reads và writes; nếu inventory được đọc ra trước rồi mới ghi, có race condition khi 2 user checkout cùng lúc cùng 1 sản phẩm. Relational ACID transaction dùng `SELECT FOR UPDATE` lock row ngay từ lúc đọc, loại bỏ race condition này hoàn toàn — đây là lý do paradigm **relational** fit với checkout flow hơn key-value.

---

## Section 3 — Deployment Evidence

> Mỗi entry = 1 screenshot (console hoặc CLI) + 1-2 dòng notes giải thích WHY.
> Thay `<!-- EDIT: ... -->` và đường dẫn ảnh bằng thật trước khi nộp.

---

### 3.1 Database — Chung (mọi engine)

#### Private Subnet

- [x] Database instance deployed trong **private subnet**, không có public IP

![DB trong private subnet](./screenshots/s3-db-private-subnet.png)
<!-- EDIT: Chụp màn hình RDS console → tab Connectivity, chỉ subnet ID + "Publicly accessible: No" -->

**Notes:** Instance `xbrain-db` nằm trong subnet `subnet-0abc1234` (10.0.3.0/24). Route table của subnet không có route `0.0.0.0/0 → igw-*`, chỉ có local route. Không có public IP được assign.
<!-- EDIT: Đổi subnet ID và CIDR thật -->

---

#### Encryption at Rest

- [x] Encryption at rest **enabled**

![Encryption at rest](./screenshots/s3-db-encryption.png)
<!-- EDIT: Chụp màn hình RDS console → tab Configuration → Storage encrypted: Yes -->

**Notes:** Storage encryption bật với AWS-managed key `aws/rds`. Chọn AWS-managed thay vì customer CMK vì chưa có compliance mandate về key rotation manual và muốn AWS tự rotate hàng năm.
<!-- EDIT: Đổi nếu dùng customer CMK hoặc engine khác -->

---

#### High Availability Plan

- [x] **Multi-AZ enabled** (managed engine) <!-- EDIT: hoặc đổi thành replica plan / SPOF acknowledged nếu self-hosted -->

![Multi-AZ](./screenshots/s3-db-multiaz.png)
<!-- EDIT: Chụp RDS console → tab Configuration → Multi-AZ: Yes -->

**Notes:** Multi-AZ enabled — synchronous standby ở `ap-southeast-1b`, primary ở `ap-southeast-1a`. Failover tự động ~60-120 giây, AWS flip CNAME endpoint, không cần thay đổi connection string ở app.
<!-- EDIT: Đổi AZ thật -->

---

#### Record Written & Read

- [x] Ít nhất **1 record được write và read** qua application hoặc CLI

![Record write/read](./screenshots/s3-db-record-rw.png)
<!-- EDIT: Chụp terminal psql hoặc app output cho thấy INSERT + SELECT -->

**Notes:** Insert 1 row vào bảng `orders` qua `psql` CLI, sau đó `SELECT` lại để verify row tồn tại với đúng giá trị. Command:
```sql
INSERT INTO orders (order_id, user_id, total, created_at)
VALUES ('ord-001', 'usr-001', 149000, NOW());

SELECT * FROM orders WHERE order_id = 'ord-001';
```
<!-- EDIT: Đổi bảng/cột/data cho đúng schema thật -->

---

### 3.2 Database — Paradigm Specific

> **Nếu chọn Relational (RDS / Aurora / self-hosted SQL):** điền section này.
> **Nếu chọn Key-Value (DynamoDB):** xóa section này, dùng section 3.2-KV bên dưới.
> **Nếu chọn Document (DocumentDB / MongoDB):** dùng section 3.2-DOC.
> **Nếu chọn Graph (Neptune):** dùng section 3.2-GRAPH.

#### [Relational] Schema — ≥2 Related Tables với Foreign Key

- [x] Schema có **ít nhất 2 bảng có quan hệ** với foreign key constraint

![Schema FK](./screenshots/s3-schema-fk.png)
<!-- EDIT: Chụp psql \d orders và \d order_items, hoặc schema diagram -->

**Notes:** Bảng `orders` (`order_id` PK) và `order_items` (`item_id` PK, `order_id` FK). Constraint:
```sql
ALTER TABLE order_items
  ADD CONSTRAINT fk_order_items_order
  FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE;
```
`ON DELETE CASCADE` đảm bảo khi order bị xóa, các item liên quan cũng bị xóa — tránh orphan rows.
<!-- EDIT: Đổi tên bảng/cột thật -->

---

#### [Relational] Indexed Lookup

- [x] Có **index hỗ trợ WHERE / JOIN** cho access pattern phổ biến

![Index](./screenshots/s3-index-lookup.png)
<!-- EDIT: Chụp EXPLAIN ANALYZE output xác nhận Index Scan -->

**Notes:** Index `idx_orders_user_id` trên `orders(user_id)`:
```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 'usr-001';
-- Output: Index Scan using idx_orders_user_id on orders  (cost=0.28..8.30 rows=3 ...)
```
Không có Seq Scan.
<!-- EDIT: Đổi index name, column, EXPLAIN output thật -->

---

#### [Relational] Automated Backups — 7+ ngày retention

- [x] Automated backups configured với **retention ≥7 ngày**

![Backup config](./screenshots/s3-backup-config.png)
<!-- EDIT: Chụp RDS console → Maintenance & backups → Automated backups: 7 days -->

**Notes:** Automated backup retention 7 ngày. Backup window: 03:00-04:00 UTC (low-traffic). Point-in-time restore available cho bất kỳ giây nào trong 7 ngày qua.
<!-- EDIT: Đổi backup window và retention thật -->

---

<!-- 3.2-KV: Uncomment section này nếu chọn DynamoDB

### 3.2-KV Database — Key-Value (DynamoDB)

#### High-Cardinality Partition Key

- [ ] Partition key là **high-cardinality** (user_id, order_id, device_id — KHÔNG phải status, active)

![Partition key](./screenshots/s3-ddb-partition-key.png)

**Notes:** Table `xbrain-orders`: PK = `user_id` (String) + SK = `order_id` (String).
`user_id` có cardinality cao (1 value per user — hàng triệu users), tránh hot partition.

#### Capacity Mode

- [ ] On-demand HOẶC Provisioned với **auto scaling enabled**

![Capacity mode](./screenshots/s3-ddb-capacity.png)

**Notes:** Chọn On-demand — tự scale theo traffic thực tế, không cần estimate capacity trước. Phù hợp với traffic pattern không đều của W3.

#### Query + GSI (không Scan)

- [ ] Primary access qua **Query** theo partition key
- [ ] Có **ít nhất 1 GSI** cho access pattern thứ 2

![Query + GSI](./screenshots/s3-ddb-query-gsi.png)

**Notes:** GSI `gsi-status-created` (PK=`status`, SK=`created_at`) phục vụ query "lấy tất cả orders có status=PENDING mới nhất". Không dùng Scan.

-->

---

<!-- 3.2-DOC: Uncomment section này nếu chọn DocumentDB / MongoDB

### 3.2-DOC Database — Document (DocumentDB / self-hosted MongoDB)

#### Aggregation Pipeline

- [ ] Có **1 aggregation pipeline** trả về kết quả shaped/grouped

![Aggregation](./screenshots/s3-doc-aggregation.png)

**Notes:** Pipeline tính tổng doanh thu theo category:
```js
db.orders.aggregate([
  { $match: { status: "COMPLETED" } },
  { $group: { _id: "$category", totalRevenue: { $sum: "$amount" } } },
  { $sort: { totalRevenue: -1 } }
])
```

#### Indexed Field Lookup

- [ ] **Secondary index** trên field thường xuyên query

![Index](./screenshots/s3-doc-index.png)

**Notes:** Index trên `user_id` để `find({ user_id: "usr-001" })` dùng IXSCAN, không COLLSCAN.
`db.orders.explain("executionStats").find({ user_id: "usr-001" })` xác nhận `winningPlan.stage: IXSCAN`.

-->

---

### 3.3 Bedrock Knowledge Base

- [x] Knowledge Base **tạo xong** và **connect với S3 bucket từ W2**

![KB created](./screenshots/s3-kb-created.png)
<!-- EDIT: Chụp Bedrock console → Knowledge Bases → Status: Active -->

**Notes:** KB `xbrain-knowledge-base` connect tới bucket `xbrain-docs-w2` (bucket từ W2, Block Public Access ON, versioning ON). Không tạo bucket mới.
<!-- EDIT: Đổi tên KB và bucket thật -->

---

- [x] **≥3 documents ingested**, sync job status: **Complete**

![Sync complete](./screenshots/s3-kb-sync.png)
<!-- EDIT: Chụp Bedrock console → KB → Data sources → Last sync: Complete + document count -->

**Notes:** Sync job hoàn thành lúc `2026-04-22 09:14 UTC`. 3 documents ingested: `policy.pdf`, `product-catalog.pdf`, `faq.md`.
<!-- EDIT: Đổi tên file và timestamp thật -->

---

- [x] **Embedding model:** Amazon Titan Embeddings G1 - Text <!-- EDIT: đổi nếu dùng model khác -->
- [x] **Vector store:** OpenSearch Serverless <!-- EDIT: đổi nếu dùng Aurora PostgreSQL hoặc S3 Vectors -->

![KB config](./screenshots/s3-kb-config.png)
<!-- EDIT: Chụp KB detail page showing embedding model + vector store -->

**Notes:** Chọn Titan Embeddings G1 - Text vì nằm trong free tier / low cost cho W3. OpenSearch Serverless là default vector store của Bedrock KB — không cần provision cluster riêng.
<!-- EDIT: Điều chỉnh reasoning cho đúng lý do thật -->

---

- [x] **1 Retrieve / RetrieveAndGenerate API call** từ Lambda hoặc CLI (không phải Playground)

![API call evidence](./screenshots/s3-kb-api-call.png)
<!-- EDIT: Chụp CloudWatch log hoặc terminal output của CLI call -->

**Notes:** Xem Section 5 — Lambda + Bedrock Evidence cho CloudWatch log và JSON response đầy đủ.

---

### 3.4 Lambda Function

- [x] Execution role **không có** `Action: "*"` hay `Resource: "*"`

![Lambda role policy](./screenshots/s3-lambda-role.png)
<!-- EDIT: Chụp IAM console → Role → Policy JSON, highlight các Action + Resource cụ thể -->

**Notes:** Role `xbrain-lambda-role` policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["bedrock:RetrieveAndGenerate", "bedrock:Retrieve"],
      "Resource": "arn:aws:bedrock:ap-southeast-1::knowledge-base/KBID123"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::xbrain-docs-w2/*"
    },
    {
      "Effect": "Allow",
      "Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"],
      "Resource": "arn:aws:logs:ap-southeast-1:123456789012:log-group:/aws/lambda/xbrain-retriever:*"
    }
  ]
}
```
Không có wildcard action hoặc resource.
<!-- EDIT: Đổi KBID, bucket name, account ID, function name thật -->

---

- [x] **Trigger** demo live: S3 event trigger <!-- EDIT: đổi thành "API Gateway integration" nếu dùng API GW -->

![Lambda trigger](./screenshots/s3-lambda-trigger.png)
<!-- EDIT: Chụp Lambda console → Configuration → Triggers -->

**Notes:** S3 trigger: bucket `xbrain-docs-w2`, event type `s3:ObjectCreated:*`, prefix `docs/`. Mỗi khi file được upload vào `docs/`, Lambda `xbrain-retriever` được invoke tự động (asynchronous invocation).
<!-- EDIT: Đổi bucket, prefix, function name thật -->

---

- [x] **Function output** visible trong CloudWatch Logs

![CloudWatch output](./screenshots/s3-lambda-cw.png)
<!-- EDIT: Chụp CloudWatch Logs → Log group → Log stream với ít nhất 1 log entry có timestamp -->

**Notes:** Xem Section 5 — Lambda + Bedrock Evidence cho log entry đầy đủ với timestamp.

---

### 3.5 VPC & Networking

- [x] VPC diagram show **3 tiers có label**: Public / Private Application / Private Database

![VPC diagram](./screenshots/s3-vpc-diagram.png)
<!-- EDIT: Paste diagram đã update từ W2, có đủ 3 tiers và S3 Gateway Endpoint -->

---

- [x] **S3 Gateway Endpoint** provision và labeled trên diagram (với route table entry)

![S3 Endpoint route table](./screenshots/s3-vpc-s3endpoint.png)
<!-- EDIT: Chụp VPC console → Route Tables → Route table của private subnet, có entry: pl-XXXXX → vpce-XXXXX -->

**Notes:** VPC Gateway Endpoint `vpce-0abc123` cho S3. Route table `rtb-private-app` có entry `pl-60b54009 (com.amazonaws.ap-southeast-1.s3) → vpce-0abc123`. S3 traffic từ private subnet không đi qua NAT Gateway — tiết kiệm cost và ở lại trong AWS backbone.
<!-- EDIT: Đổi vpce ID, rtb ID, prefix list ID thật -->

---

- [x] Database Security Group inbound rule source = **App-tier Security Group ID** (không phải CIDR)

![DB SG inbound rule](./screenshots/s3-db-sg-inbound.png)
<!-- EDIT: Chụp EC2 console → Security Groups → DB SG → Inbound rules: Source = sg-XXXXX -->

**Notes:** DB SG `sg-db-xbrain` inbound rule: Port 5432 (PostgreSQL), Source = `sg-app-xbrain` (app tier SG ID). Không dùng CIDR `10.0.2.0/24` vì SG ID là identity-based — đúng instance trong app tier mới được access, kể cả khi subnet CIDR thay đổi.
<!-- EDIT: Đổi SG IDs thật -->

---

- [x] Có thể giải thích 1 scenario khi dùng **NACL thay vì Security Group**

**Scenario:** Cần **block hoàn toàn** một dải IP bên ngoài (`203.0.113.0/24`) đang scan port 22 trên toàn subnet. Security Group chỉ có Allow rules — không thể explicit Deny. NACL có cả Allow và Deny rules, áp dụng cho toàn bộ subnet, nên có thể thêm Deny rule cho `203.0.113.0/24` ở priority cao hơn. Lưu ý: NACL stateless — phải explicit allow cả outbound return traffic trên ephemeral ports 1024-65535.
<!-- EDIT: Thay IP ví dụ, có thể dùng scenario khác phù hợp hơn với setup của nhóm -->

---

## Section 4 — Working Query Evidence

> **2 operations match với paradigm đã chọn.** Mỗi operation: command thật + screenshot output thật + ghi chú index/mechanism.
> Template mặc định: Relational. Xóa/thay nếu chọn paradigm khác.

### Operation 1 — JOIN Query (Relational)

**Mô tả:** Lấy tất cả order items của `user_id = 'usr-001'`, JOIN qua 2 bảng `orders` và `order_items`.

```sql
-- EDIT: Đổi user_id và tên bảng/cột cho đúng schema thật
SELECT
    o.order_id,
    o.created_at,
    o.total,
    oi.product_id,
    oi.quantity,
    oi.unit_price
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.user_id = 'usr-001'
ORDER BY o.created_at DESC;
```

![JOIN query result](./screenshots/s4-join-query.png)
<!-- EDIT: Chụp terminal/console với output rows thật, không phải empty result -->

**Notes:** Query dùng `idx_orders_user_id` (Index Scan trên `orders`) + Nested Loop Join với `order_items`. `EXPLAIN ANALYZE` output ở screenshot xác nhận không có Seq Scan. Execution time thực tế: ~3ms với dataset nhỏ của W3.
<!-- EDIT: Đổi execution time thật từ EXPLAIN ANALYZE -->

---

### Operation 2 — Indexed Lookup (Relational)

**Mô tả:** Lookup product detail theo `product_id` — access pattern phổ biến nhất (~200 calls/phút).

```sql
-- EDIT: Đổi product_id và tên bảng thật
EXPLAIN ANALYZE
SELECT product_id, name, price, stock_quantity
FROM products
WHERE product_id = 'prod-042';
```

![Indexed lookup + EXPLAIN](./screenshots/s4-indexed-lookup.png)
<!-- EDIT: Chụp terminal với EXPLAIN ANALYZE output + rows returned -->

**Notes:** PK lookup trên `products.product_id` (B-tree index). `EXPLAIN ANALYZE` output: `Index Scan using products_pkey on products (cost=0.28..8.30 rows=1 ...) (actual time=0.041..0.042 rows=1 ...)`. Execution time <1ms — expected cho single-row PK lookup.
<!-- EDIT: Đổi EXPLAIN output thật -->

---

<!-- Uncomment nếu chọn Key-Value (DynamoDB)

### Operation 1 — Query theo Partition Key (Key-Value)

**Mô tả:** Lấy tất cả orders của `user_id = 'usr-001'`, sort theo `created_at` mới nhất.

```python
# EDIT: Đổi table name, key values
import boto3
dynamodb = boto3.client('dynamodb', region_name='ap-southeast-1')

response = dynamodb.query(
    TableName='xbrain-orders',
    KeyConditionExpression='user_id = :uid AND created_at > :since',
    ExpressionAttributeValues={
        ':uid': {'S': 'usr-001'},
        ':since': {'S': '2026-04-01T00:00:00Z'}
    },
    ScanIndexForward=False  # DESC order
)
print(f"Items returned: {len(response['Items'])}")
print(f"Scanned count: {response['ScannedCount']}")  # phải bằng Count nếu không Scan
```

![DDB Query result](./screenshots/s4-ddb-query.png)

**Notes:** Dùng `Query` (không phải `Scan`). `ScannedCount == Count` xác nhận chỉ đọc items cần thiết — không full-table scan. Partition key `user_id` high-cardinality.

### Operation 2 — GSI Query (Key-Value)

**Mô tả:** Lấy tất cả orders có `status = 'PENDING'` mới nhất qua GSI.

```python
response = dynamodb.query(
    TableName='xbrain-orders',
    IndexName='gsi-status-created',
    KeyConditionExpression='#s = :status',
    ExpressionAttributeNames={'#s': 'status'},
    ExpressionAttributeValues={':status': {'S': 'PENDING'}},
    ScanIndexForward=False
)
```

![GSI Query](./screenshots/s4-ddb-gsi.png)

**Notes:** GSI `gsi-status-created` (PK=`status`, SK=`created_at`). Không Scan — GSI Query targeted.

-->

---

## Section 5 — Lambda + Bedrock Evidence

### Lambda CloudWatch Logs

![CloudWatch Logs](./screenshots/s5-cloudwatch-log.png)
<!-- EDIT: Chụp CloudWatch Log Group /aws/lambda/xbrain-retriever → Log stream với timestamp sau khi trigger -->

**Log entry thật (copy từ CloudWatch, đổi bằng output thật của nhóm):**
<!-- EDIT: Xóa example bên dưới và paste log thật -->
```
START RequestId: a1b2c3d4-e5f6-7890-abcd-ef1234567890 Version: $LATEST
2026-04-22T08:14:33.412Z  a1b2c3d4  INFO  S3 trigger: xbrain-docs-w2/docs/product-catalog.pdf
2026-04-22T08:14:33.891Z  a1b2c3d4  INFO  Invoking Bedrock RetrieveAndGenerate...
2026-04-22T08:14:35.102Z  a1b2c3d4  INFO  Response received. Citations: 2. Answer: 312 chars.
END RequestId: a1b2c3d4-e5f6-7890-abcd-ef1234567890
REPORT RequestId: a1b2c3d4  Duration: 1691.23 ms  Billed Duration: 1700 ms  Memory Size: 128 MB  Max Memory Used: 67 MB
```

**Notes:** Timestamp `2026-04-22T08:14:33.412Z` là sau khi upload file lên S3 lúc 08:14:30 UTC. Latency trigger → invocation: ~3 giây (async S3 trigger). Billed duration 1700ms chủ yếu là Bedrock API call (~1.5s).
<!-- EDIT: Đổi timestamp, file name, latency thật -->

---

### Bedrock RetrieveAndGenerate Response

**Gọi từ Lambda (không phải Bedrock Playground).** Output thật từ `boto3` trong Lambda function:

```json
{
  "output": {
    "text": "The return policy allows returns within 30 days of purchase with original receipt. Items must be in original condition. Digital products are non-refundable."
  },
  "citations": [
    {
      "generatedResponsePart": {
        "textResponsePart": { "span": { "start": 0, "end": 88 } }
      },
      "retrievedReferences": [
        {
          "content": { "text": "Return policy: 30 days from purchase date..." },
          "location": {
            "type": "S3",
            "s3Location": { "uri": "s3://xbrain-docs-w2/docs/policy.pdf" }
          }
        }
      ]
    }
  ],
  "sessionId": "session-abc123"
}
```
<!-- EDIT: Xóa JSON example trên và paste response thật từ Lambda/CLI của nhóm -->

![Bedrock response](./screenshots/s5-bedrock-response.png)
<!-- EDIT: Chụp terminal/CloudWatch với JSON response thật, hoặc Lambda test output -->

**Notes:** Response trích dẫn `policy.pdf` từ S3 bucket W2 — xác nhận KB đã ingest document đúng. `citations` array có `s3Location` — có thể dùng để trace nguồn câu trả lời cho user. Gọi từ Lambda function, không phải Bedrock Console Playground.
<!-- EDIT: Đổi document name, bucket thật -->

---

## Section 6 — VPC + Networking Evidence

### Route Table — S3 Gateway Endpoint

![Route table S3 endpoint](./screenshots/s6-route-table-s3-endpoint.png)
<!-- EDIT: Chụp VPC console → Route Tables → chọn route table của private-app subnet → Routes tab -->

**Entry trong route table (thay bằng thật):**

| Destination | Target | Status |
|-------------|--------|--------|
| 10.0.0.0/16 | local | Active |
| pl-60b54009 (com.amazonaws.ap-southeast-1.s3) | vpce-0abc123456 | Active |
| 0.0.0.0/0 | nat-0xyz789 | Active |

<!-- EDIT: Đổi prefix list ID, vpce ID, NAT GW ID thật -->

**Notes:** Entry `pl-60b54009 → vpce-0abc123` là S3 Gateway Endpoint. Traffic từ Lambda/app tier tới S3 đi qua VPC endpoint, không qua NAT Gateway — không tốn per-GB NAT processing fee (~$0.045/GB), và ở lại trong AWS backbone (không ra internet).
<!-- EDIT: Điều chỉnh reasoning thật -->

---

### DB Security Group Inbound Rules

![DB SG inbound](./screenshots/s6-db-sg-inbound.png)
<!-- EDIT: Chụp EC2 console → Security Groups → DB SG → Inbound rules tab -->

**Inbound rules của DB SG `sg-db-xbrain` (thay bằng thật):**

| Type | Protocol | Port | Source | Description |
|------|----------|------|--------|-------------|
| PostgreSQL | TCP | 5432 | sg-0app123 (xbrain-app-sg) | App tier access only |

<!-- EDIT: Đổi SG ID, port (3306 nếu MySQL), description thật -->

**Notes:** Source là `sg-0app123` (app-tier SG ID) — không phải subnet CIDR `10.0.2.0/24`. Dùng SG ID làm source là identity-based rule: chỉ instances có app-tier SG được access, kể cả khi IP thay đổi. CIDR-based rule sẽ cho phép bất kỳ instance nào trong subnet đó, kể cả bastion hay instance không liên quan.
<!-- EDIT: Đổi SG IDs thật -->

---

### NACL vs Security Group — Khi Nào Dùng NACL

**Scenario:** Phát hiện IP range `203.0.113.0/24` đang scan port 22 trên public subnet. Security Group chỉ có Allow rules — không thể explicit Deny một IP range. Thêm NACL Deny rule cho `203.0.113.0/24` ở priority thấp hơn (số rule nhỏ hơn) để drop traffic trước khi hit SG. NACL stateless — phải thêm cả Deny outbound cho ephemeral ports 1024-65535 từ `203.0.113.0/24` (vì return traffic cũng bị check).
<!-- EDIT: Thay bằng scenario thật từ architecture của nhóm nếu có -->

---

## Section 7 — Negative Security Test

**Kịch bản:** Thử connect trực tiếp tới RDS endpoint từ máy local (ngoài VPC) — không qua bastion, không có VPN.

```bash
# EDIT: Đổi hostname RDS endpoint thật
psql -h xbrain-db.cluster-xyz.ap-southeast-1.rds.amazonaws.com \
     -U admin \
     -d xbraindb \
     -p 5432

# Expected output (kết nối bị từ chối / timeout):
# psql: error: connection to server at "xbrain-db.cluster-xyz..." (10.0.3.45), port 5432 failed:
# Connection timed out
#         Is the server running on that host and accepting TCP/IP connections?
```

![Negative test — connection denied](./screenshots/s7-denied-connection.png)
<!-- EDIT: Chụp terminal hiện lỗi "Connection timed out" hoặc "Connection refused" -->

**Notes:** DB Security Group `sg-db-xbrain` chỉ có 1 inbound rule: port 5432 từ `sg-app-xbrain` (app-tier SG). Không có rule cho `0.0.0.0/0` hoặc bất kỳ source ngoài app-tier SG. Kết nối từ máy local bị drop silently ở SG level — "Connection timed out" (không phải "Connection refused") vì SG drop packet, không reset connection.
<!-- EDIT: Đổi SG IDs thật, có thể dùng kịch bản denied khác như unauthorized IAM action -->

---

## Section 8 — Bonus (Tùy Chọn)

> Chỉ điền nếu đã hoàn thành tất cả must-haves + Evidence Pack. Trainer chấm bonus sau khi verify sections 1-7.

<!-- EDIT: Chọn 1 scenario. Xóa comment và điền nội dung. -->

### Bonus C — Partial CloudFormation Template (+0.25)

**Scenario:** Viết partial CloudFormation template cho 1 W3 resource, validate syntax.

**Pre-state:** Không có IaC — resource được tạo thủ công qua console.

**Template (`w3-partial.yaml`):**
```yaml
# EDIT: Đổi TableName, AttributeDefinitions, KeySchema cho đúng với DynamoDB table thật
AWSTemplateFormatVersion: '2010-09-09'
Description: W3 Partial Template — DynamoDB Table xbrain-orders

Resources:
  XBrainOrdersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: xbrain-orders
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: user_id
          AttributeType: S
        - AttributeName: order_id
          AttributeType: S
      KeySchema:
        - AttributeName: user_id
          KeyType: HASH
        - AttributeName: order_id
          KeyType: RANGE
      SSESpecification:
        SSEEnabled: true
      Tags:
        - Key: Project
          Value: XBrain-W3
```

**Validate command:**
```bash
aws cloudformation validate-template --template-body file://w3-partial.yaml
```

**Expected output:**
```json
{
    "Parameters": [],
    "Description": "W3 Partial Template — DynamoDB Table xbrain-orders",
    "Capabilities": [],
    "CapabilitiesReason": ""
}
```

![CFN validate output](./screenshots/s8-cfn-validate.png)
<!-- EDIT: Chụp terminal với validate output thật -->

**Git commit:** [commit XXXXXXX](https://github.com/org/repo/commit/XXXXXXX)
<!-- EDIT: Đổi link commit thật sau khi push template lên repo -->

**Post-state:** Template đã được validate và committed vào repo. Có thể deploy bằng `aws cloudformation create-stack` hoặc qua CI/CD.

**Reflection:** Viết CFN template giúp thấy rõ structure của resource hơn so với click console — attribute types phải khai báo trước, key schema phải consistent với attribute definitions. Lần sau sẽ viết template trước rồi mới deploy thay vì làm ngược lại. `validate-template` chỉ check syntax, không check logic (ví dụ: sai region, sai ARN) — cần `create-stack` thật để catch runtime errors.
<!-- EDIT: Viết reflection thật dựa trên trải nghiệm thực tế -->

---

*Evidence Pack hoàn chỉnh. Slides thứ Sáu derive từ file này — copy 8-12 screenshots + captions vào deck và link ngược lại commit markdown này.*
<!-- EDIT: Xóa dòng này sau khi done, hoặc giữ lại như reminder -->
