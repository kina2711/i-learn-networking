# Phase 3 – TCP & Connections
## Module 01 – TCP Handshake, RTT và RTO


---


### 1. Learning Goals

Sau module này, bạn sẽ:

- Giải thích được 3‑way handshake của TCP (SYN → SYN‑ACK → ACK) bằng lời và bằng sơ đồ.
- Hiểu RTT (Round Trip Time) là gì, đo như thế nào ở mức TCP và ảnh hưởng tới latency.  
- Hiểu RTO (Retransmission Timeout) là gì, vì sao TCP phải retransmit và khi nào connection bị coi là “chết”.  
- Nhìn một lỗi `connection timeout` / `could not connect` và phân biệt được:
  - do handshake không thành công,  
  - do RTO / mạng không trả ACK,  
  - hay do app ở tầng trên.  
- Liên hệ được handshake/RTT/RTO với:
  - Database connection (Postgres/MySQL).  
  - Warehouse connection (Snowflake, BigQuery, Redshift).  
  - Kafka client ↔ broker.  
  - Spark driver ↔ executor.


---


### 2. Why It Matters For Analytics Engineers

Trong hệ thống dữ liệu, trước khi bất kỳ query / message nào được gửi đi, luôn có bước:

```text
Client mở kết nối TCP tới Service
→ Nếu handshake thất bại → không có query nào được gửi.
```

Các tình huống thực tế:

- Airflow task báo `could not connect to server: Connection timed out` khi connect Postgres.  
- BI tool lâu lâu connect Snowflake bị treo ngay từ bước mở connection, chưa kịp gửi SQL.  
- Kafka producer log `Connection to node -1 (/ip:port) could not be established. Broker may not be available.`  
- Spark driver không reach được executor vì handshake không xong → job fail ngay khi start.

Ở góc độ production:

- RTT cao → mọi query/ETL/API call đều “chậm thêm” một đoạn cố định.  
- RTO, retransmission nhiều → tăng độ trễ không rõ ràng, dễ bị nghĩ nhầm là “database chậm” hoặc “Spark chậm” trong khi gốc là mạng.

Hiểu handshake/RTT/RTO giúp bạn:

- Đặt timeout hợp lý cho JDBC/HTTP.  
- Phân biệt lỗi network vs lỗi ứng dụng.  
- Trao đổi có cơ sở với team Infra/Network khi nghi ngờ vấn đề bên dưới.


---


### 3. Intuition – TCP giống gì trong đời thường?

Hãy tưởng tượng bạn gọi điện cho người khác:

1. Bạn bấm số, chờ “reng reng” (SYN).  
2. Người kia nhấc máy và nói “alo” (SYN‑ACK).  
3. Bạn trả lời “alo” lại, bắt đầu nói chuyện (ACK).  

Nếu:

- Bạn bấm số mà không ai nhấc máy → timeout.  
- Đường dây nhiễu, một trong hai bên không nghe rõ → phải lặp lại nhiều lần (retransmission).  

Trong TCP:

- Handshake đảm bảo hai bên “nghe thấy nhau” và thống nhất số thứ tự (sequence number).  
- RTT ≈ thời gian “alo ↔ alo” qua lại.  
- RTO ≈ thời gian “đợi mãi không nghe ai trả lời” rồi kết luận “chắc mất kết nối, thử lại / bỏ luôn”.


---


### 4. TCP 3‑Way Handshake – từ SYN đến ACK

#### 4.1. Bản chất

TCP là connection‑oriented: phải thiết lập kết nối trước khi gửi dữ liệu.

Handshake 3 bước:

1. **Client → Server: SYN**  
   - “Tôi muốn mở kết nối, đây là sequence number ban đầu của tôi.”

2. **Server → Client: SYN‑ACK**  
   - “OK, tôi sẵn sàng, đây là sequence number của tôi, và tôi đã nhận SYN của bạn.”

3. **Client → Server: ACK**  
   - “Nhận SYN của bạn, từ giờ bắt đầu gửi/nhận dữ liệu.”

ASCII:

```text
Client                                         Server
  |                                              |
  | ----------- SYN (seq = x) -----------------> |
  |                                              |
  | <------ SYN-ACK (seq = y, ack = x+1) ------- |
  |                                              |
  | -------- ACK (ack = y+1) ------------------> |
  |                                              |
  |           Kết nối đã thiết lập               |
```

Sau bước này, cả hai phía đều biết sequence number của nhau, ready cho data.


#### 4.2. Nơi handshake diễn ra trong lifecycle kết nối

Ví dụ: Airflow task dùng psycopg2 connect Postgres:

```text
psycopg2.connect(...)
  ↓
OS mở socket TCP
  ↓
TCP: SYN → SYN‑ACK → ACK (3‑way handshake)
  ↓
TLS handshake (nếu dùng SSL)
  ↓
Gửi truy vấn SQL đầu tiên
```

Nếu kết nối fail sớm:

- Không phải lúc nào lỗi cũng nói “TCP handshake failed”, nhưng ở mức network đó là điều đã xảy ra.


---


### 5. RTT – Round Trip Time

#### 5.1. Định nghĩa

- RTT = thời gian để một gói tin đi từ client → server → client (một vòng đi‑về).  
- Ở mức TCP, RTT thường được ước lượng dựa trên thời gian giữa khi gửi một segment và khi nhận được ACK tương ứng.

Trong handshake:

- RTT ban đầu ≈ thời gian từ SYN đến khi nhận được SYN‑ACK, cộng thêm từ SYN‑ACK đến ACK (tùy cách đo).

#### 5.2. RTT ảnh hưởng gì?

- Độ trễ tối thiểu: mỗi request/response không thể nhanh hơn RTT (thường ít nhất 1 RTT, có khi 1.5–2 RTT tùy protocol).  
- Throughput hiệu quả của TCP phụ thuộc cả băng thông lẫn RTT: `throughput ≈ window_size / RTT`.

Ví dụ thô:

- RTT = 200ms (kết nối VN ↔ US) → mỗi request nhỏ sẽ “ăn” thêm 0.2s.  
- RTT = 5–10ms (nội bộ VPC cùng region) → phù hợp cho Kafka, Spark shuffle, DB nội bộ.

#### 5.3. Đo RTT

- Công cụ đơn giản: `ping <host>` → xem `time=... ms`.  
- Ở mức TCP, có thể dùng:
  - `tcpdump` + phân tích SYN/SYN‑ACK.  
  - Một số APM / tool network chuyên dụng.


---


### 6. RTO – Retransmission Timeout

#### 6.1. Khái niệm

- Khi TCP gửi một segment, nó đặt timer.  
- Nếu hết thời gian mà chưa nhận được ACK, nó retransmit segment đó.  
- RTO là khoảng thời gian TCP phải đợi trước khi coi như mất gói và retransmit.

RTO:

- Không cố định; được tính dựa trên RTT trung bình (SRTT) và độ biến thiên RTT.  
- Thường đủ lớn để tránh retransmit quá sớm trên mạng delay cao.

#### 6.2. Hậu quả của RTO

- Một lần RTO tối thiểu gây delay ≈ 1 giây cho luồng TCP đó.  
- Nhiều RTO liên tiếp → delay tích lũy, throughput sụt mạnh, application cảm nhận là “đơ”, “treo”.

Lý do:

- Khi bị RTO, TCP thường:
  - Giảm mạnh cửa sổ tắc nghẽn (congestion window).  
  - Quay về slow start (gửi chậm dần lên lại).  


#### 6.3. Phân biệt retransmission “bình thường” và RTO

- Retransmission do vài gói lẻ bị mất (nhưng chưa chạm RTO) → thường không quá nghiêm trọng.  
- RTO xảy ra khi quá nhiều gói bị mất / delay quá lâu, sender không thấy ACK trong thời gian RTO → mới thật sự “đau”.

Trong log / tool:

- Công cụ phân tích (Wireshark, ExtraHop, v.v.) thường phân biệt:
  - `TCP Retransmission` vs `TCP Retransmission Timeout`.


---


### 7. Reality In Production – chuyện gì xảy ra ngoài đời?

#### 7.1. Database connection timeout

Scenario:

- Airflow ở VPC A kết nối Postgres ở VPC B qua peering/VPN.  
- Route ổn, nhưng link chập chờn → nhiều packet mất trên đường.

Hậu quả:

- TCP SYN gửi đi nhưng SYN‑ACK không về kịp → client retry, tăng RTO.  
- Cuối cùng, driver/ứng dụng ném `connection timeout`.  
- Trong metric DB có thể không thấy gì, vì request chưa bao giờ tới được server.

#### 7.2. Kafka producer báo timeout

- Producer gửi batch tới broker, nhưng ACK từ broker bị mất hoặc delay.  
- TCP phải retransmit, RTO tăng → Kafka client coi request gửi thất bại, log warning/error.  
- Nếu xảy ra thường xuyên, throughput giảm, lag tăng.

#### 7.3. Spark driver ↔ executor

- Driver gửi task tới executor qua TCP.  
- Nếu một executor nằm ở node có vấn đề network (NIC lỗi, cáp lỗi, switch port flapping):
  - Nhiều retransmission, RTO.  
  - Executor có vẻ “chậm hẳn”, gây skew, stage treo.


---


### 8. Failure Scenarios – các lỗi thường gặp quanh handshake/RTT/RTO

1. Không có SYN‑ACK từ server  
   - Server không lắng nghe (service down).  
   - Firewall chặn SYN hoặc SYN‑ACK.  
   - Route tới server bị sai.  

2. RTT rất cao, nhưng vẫn kết nối được  
   - Đường mạng xa (khác continent/region) hoặc congested.  
   - Ảnh hưởng: mọi request trên kết nối đó có latency tối thiểu cao.

3. Nhiều retransmission + RTO  
   - Packet loss (cáp hỏng, port switch lỗi, radio WiFi kém).  
   - Congestion nặng ở đâu đó trên path.  

4. Half‑open connections  
   - Một bên chết không gửi FIN/RST, bên kia tiếp tục giữ connection mở.  
   - Một số DB / load balancer có idle timeout; khi client gửi lại thì gặp RST.

5. Application timeout < TCP timeout  
   - Ứng dụng timeout sớm hơn TCP (ví dụ HTTP client set 5s, TCP vẫn retry).  
   - Log app chỉ thấy “timeout” chung chung, khó phân biệt gốc TCP hay app.


---


### 9. Debug Playbook – nhìn handshake, RTT, RTO như thế nào?

- `ping <host>` – ước lượng RTT thô, kiểm tra reachable.  
- `traceroute <host>` – xem path, hop nào cao bất thường.  
- `tcpdump` với filter SYN/SYN‑ACK để xem handshake.  
- `ss` / `netstat` xem trạng thái kết nối (`SYN-SENT`, `ESTAB`, `TIME-WAIT`).  
- Wireshark hoặc tương đương để xem số lượng retransmission / RTO.


---


### 10. Data Engineering Examples

- JDBC connect Postgres thi thoảng timeout: handshake/TLS fail do loss/latency, DB log không thấy connection mới.  
- REST API trong Airflow bị “treo” 1–2s random: nhiều RTO nhỏ, cộng dồn thành delay.  
- Spark job cross‑region chậm: RTT lớn → mỗi request/response và shuffle chịu thêm delay cố định.


---


### 11. Summary

- TCP là giao thức hướng kết nối; mọi kết nối đều bắt đầu bằng three‑way handshake (SYN → SYN‑ACK → ACK).  
- RTT đo thời gian đi‑về của gói tin; là giới hạn tối thiểu của latency và ảnh hưởng trực tiếp đến throughput.  
- RTO là timeout để TCP quyết định retransmit; mỗi lần RTO tối thiểu gây delay đáng kể và thường kéo TCP quay lại slow‑start.  
- Trong hệ thống dữ liệu, rất nhiều lỗi `connection timeout` thực chất là vấn đề ở handshake/RTT/RTO, chứ không phải logic của DB/Kafka/Spark.