# Phase 3 – TCP & Connections
## Module 03 – Latency, Throughput, Packet Loss (góc nhìn của Data Platform)


---


### 1. Learning Goals

Sau module này, bạn sẽ:

- Phân biệt rõ **latency**, **throughput**, **bandwidth** và hiểu chúng liên quan thế nào trong TCP.  
- Hiểu được vì sao **mất gói rất nhỏ** vẫn có thể làm throughput giảm mạnh.  
- Biết cách **đọc và đặt SLO** cho latency (p50, p95, p99) trong hệ thống dữ liệu.  
- Liên hệ được các khái niệm này với: query DW, tải dữ liệu, Kafka lag, Spark shuffle, Airflow DAG.


---


### 2. Đặt lại định nghĩa – cho đúng từ đầu

- **Latency (độ trễ)**  
  - Thời gian từ lúc gửi request tới lúc nhận response (hoặc ACK).  
  - Đơn vị: ms, s.  
  - Bị ảnh hưởng bởi RTT, processing time, queueing, retransmission…

- **RTT (Round Trip Time)**  
  - Latency thuần về mạng cho một vòng đi‑về của gói tin (không tính processing nhiều).  
  - Thường đo bằng `ping`.

- **Throughput (thông lượng)**  
  - Lượng dữ liệu thực tế truyền thành công mỗi đơn vị thời gian (MB/s, msg/s, rows/s…).  

- **Bandwidth (băng thông)**  
  - Khả năng **lý thuyết** tối đa của đường truyền (ví dụ 1 Gbps link).  
  - Throughput ≤ Bandwidth, nhưng thường nhỏ hơn vì overhead, latency, loss, protocol…

> Hình ảnh đơn giản:  
> - Bandwidth = độ rộng ống nước.  
> - Latency = độ dài ống nước.  
> - Throughput = lượng nước thực tế bạn lấy được mỗi giây.  


---


### 3. Latency được tạo ra từ những đâu?

Độ trễ end‑to‑end của một request:

```text
Latency tổng ≈
  Network RTT (client ↔ service)
+ Queueing / waiting (LB, queue, scheduler…)
+ Processing time (DB, DW, API, Spark…)
+ Retransmission / RTO (nếu có)
```

Trong data platform:

- Query tới DW:  
  - RTT tới endpoint + thời gian planner/exec + queueing trong DW.  
- Kafka:  
  - RTT client ↔ broker + queueing + thời gian disk/replication.  
- Spark:  
  - Latency control RPC + queueing + compute.

Điểm cần nhớ:

- **RTT tạo ra một “độ trễ nền”**: mọi thứ khác còn chưa tính đã phải “chịu” ít nhất RTT, đôi khi nhiều hơn (vì nhiều round‑trip).  
- Khi **loss** tăng, latency tăng thêm do retransmission, RTO, queueing.


---


### 4. Throughput – không chỉ là “đường 1 Gbps”

Ở mức trực giác, với TCP:

```text
Throughput thực tế ≈ (lượng dữ liệu có thể outstanding trong mạng) / RTT
                   ≈ (cửa sổ TCP hiệu quả) / RTT
```

- Nếu RTT lớn nhưng cửa sổ nhỏ → throughput thấp.  
- Nếu loss khiến congestion window luôn bị kéo xuống → throughput không bao giờ “chạm” được bandwidth danh nghĩa.

Ví dụ:

- Link 1 Gbps, RTT 100ms, loss ~1%.  
- Trên lý thuyết có thể bơm 1 Gbps, nhưng do loss + congestion control, throughput thực tế có thể chỉ vài chục–trăm Mbps.


---


### 5. Packet Loss – “kẻ thù vô hình” của throughput

#### 5.1. Loss nhỏ, impact lớn

- Loss 0.0x% nghe có vẻ nhỏ, nhưng với TCP:
  - Mỗi lần loss → cwnd giảm (multiplicative decrease).  
  - Sau đó phải slow start / additive increase để build lại.  
- Nếu loss thường xuyên:
  - TCP liên tục tăng‑giảm, throughput như “răng cưa”, không bao giờ lên được cao.

#### 5.2. Loss vs Latency

- Loss không chỉ làm giảm throughput; nó còn làm tăng latency:
  - Retransmission thêm một RTT.  
  - RTO thêm khoảng delay lớn (thường ~1s trở lên).  
- Tail latency (p95, p99) rất nhạy với loss: chỉ cần một vài request dính RTO là p99 đội lên ngay.


---


### 6. Latency & Throughput trong Data Platform – các pattern thực tế

#### 6.1. Query DW nhanh/lâu tùy vị trí

- Người ở cùng region với DW: RTT nhỏ, latency tổng nhỏ hơn.  
- Người ở xa (khác châu lục): RTT lớn, cùng một query có thể chậm hơn hàng trăm ms–vài giây chỉ vì nhiều round‑trip control (auth, metadata, result…).  

=> Khi đánh giá “DW chậm”, luôn hỏi: **Client ở đâu? RTT bao nhiêu?**


#### 6.2. Tải dữ liệu lớn: throughput bị “đóng trần”

- Tải 1 TB data từ on‑prem lên cloud:  
  - Băng thông đủ, CPU/IO đủ, nhưng throughput không vượt quá X MB/s.  
  - Khi đo: RTT cao (do xuyên châu lục); một chút loss → congestion window không lên cao.

=> Cần:

- Tối ưu kích thước batch.  
- Dùng nhiều stream song song (nhưng không quá nhiều, tránh gây thêm congestion).  
- Cố gắng đặt compute gần storage (cùng region).

#### 6.3. Kafka lag vs throughput

- Producer/consumer cluster ở region này, broker ở region khác:  
  - RTT + loss → throughput “nặng” bị giảm.  
  - Lag tăng, nhưng CPU broker không hề cao.

=> Không phải Kafka “yếu”, mà là **vị trí + đặc tính mạng** không phù hợp.


#### 6.4. Spark shuffle

- Shuffle phải chuyển lượng lớn data giữa executor.  
- Nếu executor ở subnet khác / rack khác với link yếu:
  - Latency tăng, loss cao → shuffle stage kéo dài bất thường.  
- Metric CPU/IO node có thể ổn; gốc là **bottleneck network**.


---


### 7. SLO cho Latency – median vs p95/p99

Khi đặt SLO / làm dashboard:

- **p50 (median)**:  
  - Chỉ cho thấy “trung bình cuộc sống”.  
  - Không phản ánh tail khi network có vấn đề.

- **p95 / p99**:  
  - Phát hiện các outlier do RTO, loss, congestion.  
  - Quan trọng với ETL/BI: chỉ cần một phần nhỏ request rất chậm cũng đủ “đập vào mắt” user.

Ví dụ cho một API dữ liệu:

```text
SLO latency:
- p50 < 100ms
- p95 < 500ms
- p99 < 1s
```

Nếu:

- p50 ổn, p95/p99 tăng → thường là **network / queueing / loss** chứ không phải code base chậm đồng đều.


---


### 8. Đo và đọc Latency / Throughput đúng cách

#### 8.1. Đo RTT mạng

- `ping` / `mtr` tới endpoint (hoặc IP gần endpoint).  
- Ghi:

  - RTT min / avg / max.  
  - Loss %.

#### 8.2. Đo latency từ app

- Logging/metrics ở app hoặc middleware (gateway, LB):  
  - Request start → response end.  
  - Tag theo loại request (query DW, call API, insert batch…).  

- Nhìn theo phân vị (p50/p95/p99) và theo **call path** (Airflow → DW, Spark → DW, Kafka → consumer…).

#### 8.3. Đo throughput

- Đối với stream/batch:
  - MB/s, rows/s, msg/s.  
  - Break down theo link:  
    - Kafka → Spark.  
    - Spark → DW.  
    - DB → DW.

- Quan trọng: **throughput “ổn định” hay “răng cưa”**?  
  - “Răng cưa” thường gợi ý congestion/loss.


---


### 9. Failure Scenarios – từ góc nhìn Latency/Throughput

1. **BI dashboard lúc nhanh lúc cực chậm**

   - Query logic không đổi.  
   - p50 latency ổn, p95/p99 rất xấu.  

   Có khả năng:

   - Network flapping, loss tăng vào một số thời điểm.  
   - TLS handshake/connection setup chậm do RTO.  

2. **Batch load chạy lâu hơn hẳn vào giờ cao điểm**

   - Ban đêm: load 1 TB trong 30 phút.  
   - Ban ngày: cùng lượng data, mất 1.5 giờ.  

   Gợi ý:

   - Đường mạng dùng chung với traffic khác; congestion vào giờ cao điểm.  
   - TCP buộc giảm cửa sổ, throughput load giảm.

3. **Kafka lag chỉ tăng vào một số khung giờ**

   - Metric broker ổn.  
   - Lag tăng, rồi tự giảm khi qua khung giờ đó.  

   Có thể:

   - Đường liên kết giữa consumer và broker congested chung với ứng dụng khác.  


---


### 10. Exercises

1. **SLO draft cho một service dữ liệu**  
   - Chọn một service (API dữ liệu, DW query, Kafka consumer).  
   - Đề xuất:
     - SLO latency (p50/p95/p99).  
     - SLO throughput (msg/s, MB/s).  
   - Ghi rõ assumption về RTT và loss.

2. **Ping và traceroute**  
   - Ping tới:
     - Một service trong cùng region.  
     - Một service ở region xa (US vs EU vs APAC).  
   - So sánh RTT, thử dự đoán impact lên throughput.

3. **Đọc một biểu đồ “răng cưa”**  
   - Vẽ (hoặc tưởng tượng) biểu đồ throughput theo thời gian: tăng dần rồi tụt mạnh, lặp lại.  
   - Gắn mỗi phần với slow start / additive increase / multiplicative decrease.

4. **Mapping incident thật**  
   - Lấy một incident trong hệ thống: job chậm, lag Kafka, replication delay.  
   - Thử mô tả lại bằng 3 từ khóa:
     - Latency?  
     - Throughput?  
     - Packet loss / congestion?


---


### 11. Reflection Questions

1. Tại sao “thêm băng thông” (nâng từ 1 Gbps lên 10 Gbps) **không luôn** giải quyết được throughput thấp?  
2. Với workflow ETL cross‑region, bạn sẽ ưu tiên tối ưu gì: giảm số round‑trip, tăng batch size, hay tăng số luồng song song? Tại sao?  
3. Nếu bạn chỉ được có 3 metric để theo dõi cho một đường critical (ví dụ Spark → DW), bạn chọn gì và tại sao?  
4. Tail latency (p99) rất cao nhưng p50 ổn, bạn sẽ nói gì với team product / business để họ hiểu đây là vấn đề network chứ không chỉ do SQL?  
5. Trong Data Platform của bạn, link nào hiện nay có nguy cơ trở thành bottleneck về latency/throughput khi scale thêm 2–3 lần traffic?  