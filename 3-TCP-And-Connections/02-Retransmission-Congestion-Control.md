# Phase 3 – TCP & Connections
## Module 02 – Retransmission, Congestion Control, Latency & Throughput


---


### 1. Learning Goals

Sau module này, bạn sẽ:

- Hiểu được **vì sao TCP phải retransmit** và sự khác nhau giữa fast retransmit vs retransmit do timeout.  
- Nắm được trực giác về **TCP congestion control**: slow start, additive increase / multiplicative decrease (AIMD).  
- Liên hệ được **packet loss, RTT, băng thông** với throughput thực tế mà ứng dụng (DB, DW, Kafka, Spark) cảm nhận.  
- Nhìn được pattern “thi thoảng rất chậm”, “lag Kafka tăng”, “Spark shuffle nghẽn” dưới lăng kính congestion control, không chỉ nhìn CPU/IO.  


---


### 2. Intuition – Đường cao tốc giờ cao điểm

Hãy tưởng tượng:

- Mỗi packet = một chiếc xe.  
- Đường mạng = đường cao tốc.  
- Router/switch = các nút giao, trạm thu phí.  

Khi:

- Ít xe → đi thoải mái, tốc độ cao.  
- Quá nhiều xe cùng lúc → tắc, phải giảm tốc, nhiều xe “đứng chờ”.  

TCP:

- Không thấy “tắc đường” trực tiếp, chỉ thấy **mất gói** (packet loss) hoặc **RTT tăng**.  
- Khi thấy dấu hiệu tắc, nó **giảm số lượng packet đang bay trong mạng** (cửa sổ tắc nghẽn), rồi tăng dần trở lại nếu mọi thứ ổn.

Đó chính là **congestion control**.


---


### 3. Retransmission – mất gói thì làm gì?

#### 3.1. Tại sao gói bị mất?

Các nguyên nhân điển hình:

- Buffer trên router/switch đầy → drop gói.  
- Lỗi vật lý (WiFi nhiễu, cáp lỗi, port switch lỗi).  
- Queue quá dài → trễ vượt quá RTO, coi như mất.  

#### 3.2. Hai kiểu chính

1. **Fast retransmit**  
   - Receiver thấy “thiếu” một segment (dựa trên sequence number).  
   - Gửi nhiều duplicate ACK cho cùng một số thứ tự.  
   - Sender nhận đủ số duplicate ACK (thường 3) → đoán có loss → retransmit **ngay**, không đợi timeout.  

2. **Retransmit do timeout (RTO)**  
   - Không nhận được ACK nào trong thời gian RTO.  
   - Nặng hơn; thường khiến TCP “đạp phanh” mạnh: giảm cửa sổ xuống rất thấp.


---


### 4. Congestion Window & Slow Start

#### 4.1. Congestion Window (cwnd)

- `cwnd` ≈ số lượng dữ liệu tối đa mà TCP cho phép “đang bay” trong mạng mà chưa được ACK.  
- TCP phải **cân bằng**:
  - Cwnd nhỏ → không tận dụng hết băng thông.  
  - Cwnd quá lớn → router/switch không chịu nổi, bắt đầu drop.

#### 4.2. Slow Start

Khởi đầu:

- TCP không biết mạng chịu được bao nhiêu, nên **bắt đầu nhỏ**.  
- Cứ mỗi lần nhận đủ ACK, nó **nhân đôi** cwnd (tăng theo hàm mũ) cho đến khi:
  - Đạt ngưỡng (ssthresh).  
  - Hoặc bắt đầu thấy loss.

Trực giác:

```text
cwnd: 1, 2, 4, 8, 16, 32, ... (tăng rất nhanh)
```

Khi vượt qua ngưỡng:

- Chuyển sang giai đoạn tăng chậm hơn (additive increase).


---


### 5. AIMD – Additive Increase / Multiplicative Decrease

Sau slow start:

- **Increase**: mỗi RTT thành công (không thấy loss), tăng cwnd **thêm một ít** (additive).  
- **Decrease**: khi thấy loss, giảm cwnd **mạnh** (multiplicative):

Ví dụ trực giác:

```text
# Đơn giản hóa:
cwnd ban đầu: 16
Mỗi RTT "ổn": cwnd += 1  → 17, 18, 19, ...
Khi loss: cwnd := cwnd / 2  → giảm về 9, rồi tăng lại dần dần
```

Ý nghĩa:

- Tăng từ tốn để test xem mạng chịu được thêm không.  
- Khi thấy tắc → giảm mạnh để giải tỏa congestion.

> Hình ảnh quen thuộc trên dashboard: throughput tăng dần, rồi sụt mạnh, rồi tăng lại → một phần do TCP AIMD tự điều chỉnh như vậy.


---


### 6. Packet Loss, RTT, Throughput – kết nối với Data Platform

#### 6.1. Throughput ~ (cwnd / RTT)

Ở mức đơn giản:

- Cwnd càng lớn, RTT càng nhỏ → throughput càng cao.  
- Loss xuất hiện → cwnd bị kéo xuống → throughput giảm.

Trên đường:

- Cross‑region (RTT 150–200ms) + loss nhẹ → rất khó đạt throughput cao.  
- Nội bộ VPC (RTT 1–2ms) + loss gần như 0 → dễ đạt line rate.

#### 6.2. Hậu quả cho Data / Analytics

- **Kafka**:
  - Loss + RTT cao → producer/consumer khó đẩy nhanh, lag tăng, retry nhiều.  
- **Spark shuffle**:
  - Transfer khối lượng lớn giữa executor; loss khiến cwnd giảm, job kéo dài.  
- **DW load (COPY/LOAD)**:
  - Ingestion data lớn qua mạng xa; small cwnd + RTT cao → tốc độ upload không như mong đợi.  


---


### 7. Reality In Production

#### 7.1. “Network xấu nhẹ” – khó phát hiện

- Loss 0.1–0.5%:
  - Không làm gãy kết nối.  
  - Nhưng khiến TCP phải cứ tăng rồi giảm cwnd liên tục → throughput thấp hơn lý thuyết rất nhiều.  
- Trên log app:
  - Không luôn thấy error rõ ràng.  
  - Chủ yếu là “random slow”, độ trễ cao ở tail (p95, p99).

#### 7.2. “Burst loss” – RTO, timeout

- Đợt congestion lớn hoặc switch reboot: nhiều gói liên tiếp bị mất.  
- TCP không nhận được ACK trong RTO → timeout, retransmit, giảm cwnd xuống rất nhỏ.  
- Ứng dụng:
  - Thấy spike latency, error, connection reset.  
  - Kafka có thể báo `request timeout`, Spark stage fail, Airflow task timeout.


---


### 8. Failure Scenarios – pattern bạn sẽ gặp

1. **Upload dữ liệu lên DW không bao giờ đạt quá X MB/s**  
   - Dù băng thông danh nghĩa cao hơn nhiều.  
   - Trên link cross‑region, RTT lớn + một chút loss → TCP luôn bị giới hạn bởi cửa sổ.  

2. **Kafka lag tăng vào giờ cao điểm**  
   - Khi traffic tăng, router/switch congested → loss tăng.  
   - TCP giảm cwnd → producer/consumer không kịp đẩy, lag tăng dù cluster Kafka không quá tải CPU/IO.

3. **Spark job chạy “hên xui”**  
   - Cùng job, cùng data, lúc chạy nhanh, lúc lâu bất thường.  
   - Khi check network monitoring mới thấy một số thời điểm loss tăng, liên quan đến slow stage.

4. **DB replication giữa region**  
   - Replication bị delay, hoặc thỉnh thoảng tụt xa.  
   - Link WAN giữa DC có loss/latency khiến throughput replication không đạt → backlog.


---


### 9. Debug Playbook – khi nghi ngờ congestion / loss

**1. Đo RTT & loss dài hạn**

- Dùng `mtr`, `ping` dài để:
  - Quan sát RTT min/avg/max.  
  - Xem loss theo hop.  

**2. Xem metric network**

- Ở cloud / datacenter:
  - Error/drop trên interface NIC.  
  - Congestion / queue length ở router, load balancer.  

**3. Capture traffic**

- `tcpdump` + Wireshark:
  - Đếm số `retransmission`, `fast retransmission`, `dup ACK`.  
  - Xem pattern cwnd tăng/giảm (Wireshark có graph).

**4. So sánh app metric vs network metric**

- Nếu CPU/IO DB, Kafka, Spark đều bình thường nhưng:
  - Tail latency cao, throughput thấp, nhiều retry → rất có thể là network/congestion.  


---


### 10. Data Engineering Examples

**Example 1 – Kafka ingestion vào DW qua region khác**

- Pipeline: Kafka (region A) → Spark job (region A) → DW (region B).  
- Đến một ngưỡng, tăng số partition/worker không cải thiện throughput nữa.

Giải thích:

- Cross‑region RTT lớn, một phần loss → TCP từ Spark tới DW bị giới hạn bởi cwnd.  
- Dù bạn có tăng song song, mỗi stream vẫn bị chặn bởi cùng đặc tính TCP.

**Example 2 – Airflow call REST API external**

- Giờ cao điểm, nhiều HTTP call từ Airflow tới API bên ngoài.  
- Log: timeout tăng, retry nhiều; API team nói không thấy CPU tăng.

Có thể:

- Link ra Internet congested; loss tăng → TCP giảm cwnd, RTO tăng.  
- App chỉ thấy “API chậm”, nhưng gốc là congestion ở outbound link / proxy.

**Example 3 – Spark shuffle trong cluster on‑prem**

- Một rack switch lỗi, buffer nhỏ → dễ full khi nhiều job chạy.  
- Node gắn vào switch đó trở thành “nút cổ chai”: packet loss → cwnd luôn nhỏ → mọi shuffle đi qua node đó bị chậm.


---


### 11. Exercises

1. **Đọc một biểu đồ throughput “răng cưa”**  
   - Tự vẽ một đồ thị tưởng tượng: throughput tăng dần rồi tụt, lặp lại.  
   - Gắn từng giai đoạn với slow start, additive increase, multiplicative decrease.

2. **Thử mtr tới một endpoint xa (ví dụ region khác)**  
   - Ghi lại RTT, loss trung bình.  
   - Dự đoán xem khi upload 1 TB data, throughput tối đa thực tế có thể bị giảm như thế nào so với lý thuyết.

3. **Lấy một incident thật trong hệ thống**  
   - Ví dụ: Kafka lag tăng, Spark job chậm, DB replication delay.  
   - Viết lại dưới hai góc nhìn:
     - “Tool X chậm”.  
     - “Đường mạng có congestion / loss, TCP buộc phải giảm cửa sổ”.

4. **Timeout tuning experiment**  
   - Chọn một API/DB bạn có thể test.  
   - Chạy cùng request với nhiều cấu hình timeout/retry khác nhau.  
   - Quan sát khi network bị cố tình “làm xấu” (dùng `tc`):  
     - Value nào cho trải nghiệm ổn nhất (ít fail, không treo quá lâu).


---


### 12. Reflection Questions

1. Nếu bạn chỉ monitor **băng thông sử dụng (Mbps)** mà không monitor RTT/loss, bạn có bỏ sót vấn đề gì liên quan đến congestion?  
2. Trong Kafka, tại sao lag có thể tăng dù broker CPU/IO thấp và traffic trung bình không thay đổi nhiều?  
3. Đối với Spark, khi nào bạn nên nghi ngờ network/congestion thay vì blame code/query?  
4. Với một pipeline cross‑region, bạn sẽ thiết kế batch size, concurrency, timeout như thế nào để “thân thiện với TCP”?  
5. Bạn có metric / alert nào để phát hiện “network xấu nhẹ nhưng dai dẳng” trước khi nó trở thành incident lớn không?  