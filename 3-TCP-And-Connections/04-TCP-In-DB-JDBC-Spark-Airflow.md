# Phase 3 – TCP & Connections
## Module 04 – TCP trong Database, JDBC, Spark, Kafka, Airflow


---


### 1. Learning Goals

Sau module này, bạn sẽ:

- Nhìn thấy rõ **kết nối TCP** ẩn dưới mỗi kết nối DB/JDBC, Kafka, Spark, Airflow.  
- Hiểu lifecycle điển hình: DNS → TCP handshake → TLS (nếu có) → protocol app (SQL, Kafka, HTTP…).  
- Biết các loại **timeout / error** thường gặp và mapping chúng về: DNS, TCP, TLS, ứng dụng.  
- Có checklist tối thiểu để debug:
  - DB connection issue.  
  - Kafka broker không reachable.  
  - Spark driver không nói chuyện được với executor.  
  - Airflow task bị timeout khi gọi DB/API.


---


### 2. Từ góc nhìn TCP: mọi “connection string” chỉ là IP:Port

#### 2.1. Connection string DB

Ví dụ PostgreSQL:

```text
postgresql://user:pass@orders-db.prod.internal:5432/orders
```

Đằng sau:

1. DNS: resolve `orders-db.prod.internal` → IP.  
2. TCP: client mở kết nối tới `IP:5432` (3‑way handshake).  
3. TLS (nếu `sslmode=require`): TLS handshake trên TCP.  
4. Application protocol: gói “startup” của Postgres, auth, rồi SQL.

Nếu fail:

- DNS sai → không tới được IP.  
- TCP handshake fail → `connection timed out`, `connection refused`.  
- TLS fail → `SSL handshake failed`.  
- App fail → `authentication failed`, `relation does not exist`, v.v.


#### 2.2. Connection string Kafka

```text
bootstrap.servers=broker-1.kafka.prod.internal:9092,...
```

Luồng:

1. DNS resolve từng hostname broker.  
2. TCP kết nối tới `IP:Port` của broker.  
3. TLS/SASL (nếu bật).  
4. Kafka protocol.

Thông điệp:

> Trước khi blame “Kafka”, “Postgres”, “Snowflake”, hãy tự hỏi:  
> “Kết nối TCP tới IP:Port của nó đã thành công chưa?”


---


### 3. Database & JDBC – bên dưới một query SQL

#### 3.1. Connection pool và TCP

Connection pool (HikariCP, pgBouncer, DataSource…) thực chất quản lý **nhiều TCP connection** tới DB:

- Mỗi connection = 1 cặp TCP (client port ↔ server port 5432/3306…).  
- Pool giữ connection **ESTABLISHED** để tái sử dụng, tránh phải handshake/TLS cho mỗi query.

Flow cơ bản:

```text
App Thread
  ↓  (mượn connection từ pool)
TCP (đã handshake sẵn, có TLS)
  ↓
Gửi SQL → Nhận kết quả
  ↓
(trả connection về pool)
```

Nếu TCP bị reset / timeout:

- Pool phát hiện connection chết → loại bỏ, mở connection mới (TCP handshake mới).  
- Trong code, thường thấy exception JDBC “connection closed”, “broken pipe”, “read timeout”…


#### 3.2. Các loại timeout & error thường gặp

- DNS:
  - `Unknown host`, `could not translate host name`.  
- TCP:
  - `Connection timed out` (không có SYN‑ACK).  
  - `Connection refused` (port không lắng nghe hoặc firewall reset).  
- TLS:
  - `SSL handshake failed`, `certificate verify failed`.  
- App/DB:
  - `password authentication failed`, `query canceled`, `lock timeout`.

Checklist khi DB connection issue:

1. Từ host chạy app:  
   - `dig` / `nslookup` hostname DB.  
   - `ping` hoặc `traceroute` tới IP.  
   - `nc -vz host port` để thử mở TCP.  
2. Nếu TCP OK:
   - Kiểm tra TLS (dùng `openssl s_client -connect host:port`).  
   - Kiểm tra auth/role/connection limit trong DB.  


---


### 4. Spark – driver, executor và TCP

#### 4.1. Control plane: driver ↔ executor

Spark sử dụng TCP để:

- Driver gửi task, nhận status từ executor.  
- Executor gửi log, metric, shuffle metadata… về driver.

Trong cluster:

- Mỗi executor có host/IP/port.  
- Driver giữ một **bảng connection** tới các executor.

Nếu TCP giữa driver và executor có vấn đề:

- Executor không register được với driver.  
- Task submit bị treo, job fail với lỗi kiểu `Lost executor`, `Stage cancelled`.

#### 4.2. Shuffle & data transfer

- Dữ liệu shuffle giữa executor cũng đi qua TCP (thường là HTTP/Netty, nhưng vẫn dựa trên TCP).  
- Độ trễ và loss trên mạng nội bộ cluster trực tiếp ảnh hưởng tới:
  - Thời gian stage shuffle.  
  - Skew (một số node chậm hẳn).  

Checklist khi spark job fail “mơ hồ”:

- Node bị lose connection nhiều → check network (packet loss, NIC, cáp, port).  
- RTT nội bộ cluster có spike bất thường không.  
- Có firewall / security group nào vừa thay đổi block port của Spark?


---


### 5. Kafka – TCP trong producer/consumer

#### 5.1. Producer → Broker

Flow:

1. DNS resolve hostname bootstrap/broker.  
2. TCP handshake tới broker.  
3. TLS/SASL (nếu có).  
4. Gửi metadata request để biết list broker/partition.  
5. Gửi batch message, nhận ACK.

Nếu TCP không ổn:

- Log producer:  
  - `Connection to node X could not be established`.  
  - `NetworkException`, `TimeoutException`.  
- Lag tăng, retry nhiều.

#### 5.2. Consumer ↔ Broker

Consumer group cần giao tiếp đều đặn với broker để:

- Fetch batch message.  
- Commit offset.  
- Gửi heartbeat.

Nếu TCP bị flapping:

- Consumer bị “kick ra” group.  
- Rebalance liên tục, lag nhảy lung tung.


Checklist:

- Kiểm tra `nc -vz broker port` từ host client.  
- Coi log có pattern `connection reset`, `timed out`, `unreachable`.  
- Đo RTT / loss giữa client & broker, nhất là nếu cross‑region / cross‑VPC.


---


### 6. Airflow – orchestration nhưng lỗi lại là network

#### 6.1. Airflow task & TCP

Mỗi loại task tương ứng với một loại kết nối:

- `PostgresOperator`, `MySqlOperator`, `SnowflakeOperator`… → TCP tới DB/DW.  
- `SimpleHttpOperator`, `HttpSensor` → TCP (HTTP/HTTPS) tới API.  
- `SSHOperator` → TCP 22.  
- Hook (S3, GCS, BigQuery…) → HTTP/HTTPS, vẫn là TCP.

Tức là: **mỗi task là một client TCP nhỏ**.

#### 6.2. Pattern lỗi phổ biến

- Lỗi DNS:
  - `could not translate host name`, `Name or service not known`.  
- Lỗi TCP:
  - `Connection refused`, `Connection timed out`.  
- Lỗi TLS:
  - `SSL: CERTIFICATE_VERIFY_FAILED`.  
- Lỗi HTTP:
  - `Read timed out`, `Max retries exceeded with url`.

Checklist debug một task Airflow fail vì network:

1. `kubectl exec` / SSH vào chính worker/container chạy task.  
2. Từ đó, chạy:
   - `dig` hostname.  
   - `ping`, `traceroute` tới IP/hostname.  
   - `nc -vz host port` hoặc `curl -v https://host:port`.  
3. So sánh với hành vi từ laptop:  
   - Nếu laptop OK, worker không OK → vấn đề ở VPC/subnet/security group/DNS của cluster.


---


### 7. Mapping error message → layer

Bảng cheat nhỏ:

| Error / Message (ví dụ) | Nhiều khả năng thuộc layer nào? |
| ------------------------ | -------------------------------- |
| `could not translate host name`, `Unknown host` | DNS |
| `Connection timed out` (ngay lúc connect) | TCP (handshake không xong) |
| `Connection refused` | TCP (port không lắng nghe / firewall reset) |
| `SSL handshake failed`, cert error | TLS |
| `socket timeout`, `Read timed out` (sau khi gửi request) | TCP (RTO/retransmit) hoặc app chậm |
| `HTTP 4xx/5xx`, `SQL error` | Application (logic, auth, validation…) |

Khi đọc log:

1. Nhìn keyword, map nhanh vào **DNS / TCP / TLS / APP**.  
2. Chọn đúng “hộp đồ nghề”:
   - DNS → `dig`, `nslookup`.  
   - TCP → `ping`, `traceroute`, `nc`, `tcpdump`.  
   - TLS → `openssl`, `curl -v`.  
   - APP → log/metric app, query plan, code.


---


### 8. Debug Playbook tổng hợp cho Data Engineer

Khi có lỗi “cannot connect” hoặc “timeout” trong DB/Kafka/Spark/Airflow:

1. **Check DNS**  
   - `dig` hostname từ đúng environment.  
   - Nếu fail → fix DNS trước.

2. **Check TCP**  
   - `ping` (nếu ICMP không bị chặn, chỉ để ước lượng RTT).  
   - `traceroute` để xem path.  
   - `nc -vz host port` để verify mở được TCP.

3. **Check TLS (nếu dùng HTTPS/SSL)**  
   - `openssl s_client -connect host:port`.  
   - Xem cert, SNI, lỗi chain.

4. **Check application**  
   - Log DB (authentication, max connections).  
   - Log Kafka (broker down, auth, quota).  
   - Log Spark/Airflow.

5. **Khi nghi ngờ deeper network**  
   - `tcpdump` trên client/server.  
   - Tìm SYN không được trả lời, RST, RTO, retransmission.


---


### 9. Exercises

1. **Vẽ lại lifecycle một DB connection**  
   - Chọn Postgres/MySQL ở môi trường bạn đang dùng.  
   - Vẽ các bước: DNS → TCP → TLS → auth → query.  
   - Ghi ví dụ error ở từng bước.

2. **Hands‑on nhỏ với `nc` và `tcpdump`**  
   - Từ một host, dùng `nc -vz host port` tới DB/Kafka/DW.  
   - Chạy `tcpdump` song song để xem SYN/SYN‑ACK/ACK.

3. **Phân loại lỗi từ log thật**  
   - Lấy 5–10 log error gần đây (Airflow, Spark, Kafka, app).  
   - Với mỗi log, gắn tag: DNS / TCP / TLS / APP.  
   - Viết 1–2 câu “nếu gặp lại lỗi này, mình sẽ check những gì đầu tiên?”.

4. **Thiết kế “network section” cho runbook incident**  
   - Cho một hệ thống (VD: “Airflow → Kafka → Spark → Warehouse”).  
   - Viết phần runbook mô tả:
     - Lệnh kiểm tra nhanh (DNS, TCP, TLS).  
     - Chỉ số network cần xem (RTT, loss, retransmission).  


---


### 10. Reflection Questions

1. Trong stack mà bạn đang vận hành, component nào nhạy cảm với latency nhất (BI, API, Kafka, Spark, DW…)? Vì sao?  
2. Khi design một service mới (ví dụ API phục vụ BI), bạn sẽ đặt timeout/connect‑timeout như thế nào để cân bằng giữa “fail nhanh” và “chịu được network hơi xấu”?  
3. Bạn đã từng gặp incident mà ban đầu blame “database chậm”, nhưng RCA cuối cùng lại là network/TCP? Nếu có, hãy thử map lại theo các bước trong module này.  
4. Với mỗi loại tool (DB, Kafka, Spark, Airflow), bạn có 3 lệnh CLI nào “quen tay” để check DNS/TCP/TLS? Nếu chưa có, hãy định nghĩa bộ 3 đó sau module này.  