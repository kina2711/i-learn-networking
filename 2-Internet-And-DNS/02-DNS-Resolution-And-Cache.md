# Phase 2 – Internet & DNS
## Module 04 – DNS trong Data Platform (DB, Warehouse, Kafka, Cloud)


---


### 1. Mục tiêu của module

Sau khi hoàn thành module này, người học:

- Hiểu được vai trò trung tâm của DNS trong kiến trúc Data Platform: từ database, data warehouse, Kafka đến microservice / API.  
- Phân biệt được các kiểu DNS dùng trong môi trường cloud: public DNS, private DNS, split‑horizon, cloud‑provided resolver.  
- Mô tả được cách thiết kế hostname / domain cho database và dịch vụ dữ liệu (prod/stg/dev, region, VPC…).  
- Nhận diện được các pattern lỗi DNS đặc trưng trong Data Platform (DB failover, Kafka broker name, private endpoint, cross‑VPC…).  
- Có “checklist DNS” khi thiết kế/triển khai một data service mới (DB, warehouse, Kafka, Spark service…).


---


### 2. Tại sao DNS là “xương sống” của Data Platform?

Trong một Data Platform thực tế, gần như mọi thành phần đều nói chuyện với nhau qua hostname:

- Database: `orders-db.prod.internal`, `mydb.region.rds.amazonaws.com`.  
- Data warehouse: endpoint Snowflake/BigQuery/Redshift/Databricks.  
- Kafka brokers: `broker-1.kafka.cluster.local`, `b-1.mykafka.amazonaws.com`.  
- Service / API: `metrics.service.svc.cluster.local`, `analytics-api.prod.example.com`.  

Hệ quả:

- Thay đổi IP của bất kỳ thành phần nào (scale, failover, move region) có thể được che giấu thông qua DNS – ứng dụng chỉ cần hostname ổn định.  
- Ngược lại, nếu DNS sai / inconsistent, cả cluster có thể “tự dưng chết” dù service vẫn chạy tốt từng node.


---


### 3. DNS cho Database (PostgreSQL / MySQL / Cloud SQL / RDS)

#### 3.1. Vì sao luôn nên dùng hostname cho DB?

- Các dịch vụ DBaaS trên cloud (RDS, Cloud SQL, Exadata Cloud, v.v.) đều khuyến nghị truy cập DB bằng hostname, không dùng IP thô.  
- Khi failover / switchover, provider sẽ:
  - Cập nhật DNS để hostname trỏ sang node mới.  
  - Ứng dụng không cần đổi cấu hình, chỉ cần tôn trọng TTL.

Ví dụ:

- Endpoint RDS: `mydb.abc123.rds.amazonaws.com`.  
- Custom DNS cho Cloud SQL: `mydb.sql.example.internal` (qua Cloud DNS / Private DNS).  

#### 3.2. Kiểu hostname cho DB nội bộ

Gợi ý naming:

```text
<service>-db.<env>.<region>.<domain>

# Ví dụ:
orders-db.prod.ap-southeast1.internal.example.com
```

Lợi ích:

- Nhìn hostname là biết: service nào, môi trường nào (prod/stg/dev), region nào, domain nào.  
- Dễ tạo rule DNS/Firewall theo pattern.

> Một thói quen tốt: khi vẽ sơ đồ Data Platform, luôn ghi rõ hostname DB/Warehouse, không chỉ vẽ một ô “Postgres” hoặc “Snowflake” chung chung.


---


### 4. DNS cho Data Warehouse / Lakehouse

#### 4.1. Endpoint warehouse thường là CNAME động

- Snowflake, BigQuery, Redshift, Databricks… đa số cấp endpoint hostname chứ không đưa IP trực tiếp, phía sau là cả một lớp routing / load‑balancing.  
- Ví dụ (minh họa):
  - `account.region.snowflakecomputing.com` → CNAME → nhiều A record.  
  - `region.bigquery.googleapis.com` → do Cloud DNS quản lý.  

Ý nghĩa:

- Provider toàn quyền thay đổi IP, topology backend, load balancer mà không ảnh hưởng client.  
- Có thể route theo region, nearest POP, trạng thái health backend.

#### 4.2. Private endpoint, Private DNS

- Nhiều DW hỗ trợ private endpoint trong VPC, chỉ expose qua private DNS:
  - Ví dụ: `mydw.privatelink.region.cloudprovider.com` được resolve thành IP private trong VPC.  
- Khi đó:
  - Ứng dụng trong VPC dùng resolver của VPC để thấy IP private.  
  - Ứng dụng ngoài VPC có thể thấy IP khác, hoặc NXDOMAIN.

Checklist khi debug:

- Chạy `dig`/`nslookup` từ:
  - Container/pod/VM trong VPC.  
  - Laptop/VM ngoài VPC.  
- So sánh:
  - Kết quả có khác nhau không (split‑horizon DNS)?  
  - IP trả về có thuộc dải subnet private mong đợi không?


---


### 5. DNS cho Kafka và các dịch vụ streaming

#### 5.1. Hostname broker và listener

Kafka client thường cấu hình bằng hostname broker:

```text
bootstrap.servers=broker-1.kafka.prod.internal:9092,broker-2.kafka.prod.internal:9092
```

Hoặc endpoint managed service:

```text
b-1.mykafka.abc123.kafka.region.amazonaws.com:9094
```

Lưu ý:

- Client cần resolve được hostname của từng broker, không chỉ hostname bootstrap.  
- Trong setup phức tạp (SASL_SSL, nội bộ + Internet), DNS có thể trả IP khác nhau theo mạng.

#### 5.2. Pattern lỗi DNS với Kafka

Các lỗi hay gặp:

- Lúc connect được, lúc không:
  - Một số broker name resolve fail (NXDOMAIN) hoặc trả IP unreachable.  
- Khi migrate cluster / đổi domain:
  - Consumer/producer cũ vẫn cache hostname/IP, gây lỗi kết nối.

Checklist:

- Dùng `dig`/`nslookup` kiểm tra tất cả hostname trong `bootstrap.servers` từ chính host chạy client.  
- Kiểm tra:
  - Có CNAME chain không?  
  - TTL bao nhiêu?  
  - IP trả về có thuộc đúng VPC/subnet không?


---


### 6. DNS trong Kubernetes / Service Mesh

#### 6.1. Cluster DNS

Trong Kubernetes:

- Service được resolve bằng tên:  
  - `my-service` (trong cùng namespace).  
  - `my-service.my-namespace.svc.cluster.local`.  
- CoreDNS/kube-dns chịu trách nhiệm:
  - Trả IP ClusterIP cho service.  
  - Forward truy vấn ra ngoài (external DNS, VPC DNS).

Đối với Data Platform:

- Spark driver, Airflow worker, Kafka client chạy trong pod dùng cluster DNS để gọi DB, API nội bộ.  
- Các kết nối ra ngoài (DW, SaaS) phụ thuộc cấu hình forwarder của cluster DNS.

#### 6.2. Các lỗi thường gặp

- Sai tên service / namespace → NXDOMAIN trong CoreDNS.  
- Thiếu search domain → `service-b` không được expand thành `service-b.ns.svc.cluster.local`.  
- CoreDNS cấu hình sai forwarder hoặc bị network policy chặn.

Giải pháp:

- `kubectl exec` vào pod → `dig`/`nslookup` hostname cần gọi.  
- Kiểm tra `cat /etc/resolv.conf`.  
- Xem log, metrics của CoreDNS.


---


### 7. Public DNS vs Private DNS vs Split‑Horizon

#### 7.1. Public DNS

- Dùng cho:
  - Endpoint public (console, public API, public website).  
  - BI/UI truy cập từ Internet.

#### 7.2. Private DNS / VPC DNS

- Chỉ visible từ:
  - VPC nội bộ.  
  - On‑prem qua VPN/Direct Connect/Interconnect.  
- Dùng cho:
  - Database nội bộ, internal API, Kafka cluster, Redis, cluster Spark, v.v.  

#### 7.3. Split‑Horizon DNS

- Cùng một tên, nhưng:
  - Từ Internet: IP public.  
  - Từ VPC/on‑prem: IP private.  
- Hữu ích cho:
  - Deploy hybrid (on‑prem + cloud).  
  - Giữ một hostname duy nhất cho DB/Warehouse, nhưng route khác nhau tuỳ nơi gọi.

> Câu cần nhớ: “Cùng hostname, khác câu trả lời DNS” là chuyện bình thường khi dùng split‑horizon – đừng vội nghĩ DNS “sai”.


---


### 8. DNS trong kiến trúc hybrid & multi‑cloud

#### 8.1. Hybrid: On‑prem ↔ Cloud

- On‑prem có DNS nội bộ (BIND, AD DNS, Infoblox…).  
- Cloud có DNS riêng (Route 53, Azure DNS, Cloud DNS, Private DNS của các provider).  
- Thường cần:
  - Forwarder / conditional forwarder hai chiều.  
  - Private Resolver (Azure DNS Private Resolver, OCI Private DNS, các cơ chế tương đương).

Use case điển hình:

- App on‑prem gọi DB trên cloud bằng hostname private.  
- App trên cloud gọi Kafka/DB on‑prem bằng hostname on‑prem.

#### 8.2. Multi‑cloud

- Mỗi cloud có DNS service riêng, nhưng phải phối hợp qua:
  - Public DNS (domain chính).  
  - Hoặc DNS trung tâm (3rd‑party, on‑prem).  

Đối với Data Platform:

- Data warehouse trên cloud A, Kafka trên cloud B, BI tool on‑prem:
  - DNS là glue kết nối tất cả – nếu thiết kế kém, multi‑cloud sẽ rất khó vận hành.


---


### 9. Checklist DNS khi thiết kế một data service

Khi thêm một thành phần (DB, DW, Kafka, API, Spark service) mới, tự hỏi:

1. **Hostname là gì?**  
   - Theo naming convention nào (service, env, region, domain)?  

2. **DNS zone ở đâu?**  
   - Public hay private?  
   - Managed bởi ai (cloud DNS, on‑prem, bên thứ ba)?  

3. **Client nào sẽ gọi service này?**  
   - On‑prem, cloud A, cloud B, Internet?  
   - Mỗi nơi có resolver nào, nhìn thấy zone nào?  

4. **TTL hiện tại là bao nhiêu?**  
   - Có kế hoạch đổi IP / failover thường xuyên không?  
   - TTL cần thấp hay cao?  

5. **Đã test từ môi trường “giống production” chưa?**  
   - Không chỉ test `dig` từ laptop; phải test từ pod/VM/worker thật.  


---


### 10. Tóm tắt

- DNS là lớp “dán” các thành phần của Data Platform lại với nhau: DB, DW, Kafka, Spark, Airflow, API, Kubernetes, on‑prem, cloud.  
- Các khái niệm như private DNS, split‑horizon, VPC resolver, custom resolver… là công cụ để triển khai kiến trúc hybrid/multi‑cloud một cách sạch, ổn định.  
- Lỗi DNS trong Data Platform thường không chỉ là “hostname sai”, mà còn đến từ:
  - Khác biệt resolver / zone giữa on‑prem, VPC, cluster.  
  - TTL & cache, failover, multi‑region, multi‑cloud.  
- Với tư duy đúng, DNS trở thành đòn bẩy kiến trúc (service discovery, routing, failover), không chỉ là “thứ bắt buộc phải có”.