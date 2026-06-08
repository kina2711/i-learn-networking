# Phase 1 – Networking Fundamentals
## Module 04 – Từ Laptop tới Internet


---


### 1. Mục tiêu của module

Sau khi hoàn thành module này, người học:

- **Mô tả được** đường đi end-to-end của một gói tin từ laptop (hoặc máy dev) ra Internet và tới dịch vụ trên cloud.
- **Hiểu được** vai trò của: card mạng, IP cục bộ, default gateway, NAT, ISP, router biên, load balancer, VPC…
- **Nhận diện được** các điểm có khả năng gây lỗi trên đường đi này (DNS, WiFi, NAT, firewall, route, proxy…).
- **Áp dụng được** các lệnh cơ bản (`ipconfig`/`ifconfig`, `ip route`, `ping`, `traceroute`, `curl`, `dig`) để kiểm tra từng đoạn.
- **Liên hệ được** đường đi từ laptop tới Internet với các công việc hàng ngày: truy cập console cloud, kết nối tới data warehouse, SSH vào bastion, gọi API phục vụ data pipeline.


---


### 2. Bức tranh tổng quan – từ laptop tới dịch vụ trên cloud

Ở mức cao, khi một ứng dụng trên laptop gọi tới một dịch vụ trên cloud (ví dụ web UI của BigQuery, Snowflake, Databricks, API nội bộ…), luồng điển hình như sau:

```mermaid
flowchart LR
    A[Laptop<br/>(BI, Browser, CLI...)]
    B[WiFi Router / Switch<br/>(LAN)]
    C[Router/NAT của modem]
    D[ISP / Nhà mạng]
    E[Internet Backbone]
    F[Cloud Edge Router]
    G[Cloud Load Balancer / API Gateway]
    H[VPC / Private Network<br/>(Service, DB, DW...)]

    A --> B --> C --> D --> E --> F --> G --> H
```

Từng hop tương ứng với:

- **A → B**: từ laptop tới switch hoặc access point WiFi trong LAN.  
- **B → C**: trong mạng nội bộ nhà / công ty, tới router làm gateway.  
- **C → D → E**: qua router/NAT tới nhà mạng, đi trên hạ tầng Internet.  
- **F → G → H**: đi vào hạ tầng cloud / data center, qua edge router, load balancer, rồi vào VPC/private network chứa dịch vụ.

> [!IMPORTANT]
> Khi debug, có thể xem mỗi mũi tên trên như **một đoạn cần kiểm tra riêng**: nếu biết đoạn nào đang hỏng, phạm vi tìm lỗi sẽ thu hẹp rất nhiều.


---


### 3. Bước 1 – Bên trong laptop

#### 3.1. Tầng ứng dụng và TCP/IP stack

Quy trình cơ bản:

1. Ứng dụng (browser, BI tool, psql, ssh, curl…) tạo request.  
2. Hệ điều hành dùng **DNS** để tìm IP đích (nếu chỉ biết hostname).  
3. TCP/IP stack trên máy thiết lập kết nối TCP (hoặc UDP) đến IP/port đích.  
4. Dữ liệu được đóng gói theo mô hình TCP/IP và OSI, gửi xuống card mạng (NIC).

Các thông tin quan trọng trên laptop:

- Địa chỉ IP cục bộ (IPv4/IPv6).  
- Default gateway (IP của router trong LAN).  
- DNS server đang sử dụng.  
- Bảng định tuyến (routing table).

Lệnh kiểm tra thường dùng:

```bash
# Windows
ipconfig /all

# Linux / macOS
ip addr show         # xem IP
ip route             # xem route & default gateway
cat /etc/resolv.conf # xem DNS
```

> [!TIP]
> Nếu IP, gateway hoặc DNS trên chính laptop đã sai, mọi thứ tiếp theo đều không ổn. Luôn bắt đầu debug từ đây.


---

### 4. Bước 2 – Từ laptop tới mạng LAN (Switch / WiFi AP)

#### 4.1. Kết nối Layer 2: MAC, ARP và switch

Khi gửi gói tin ra ngoài subnet:

- Laptop cần gửi gói IP tới **default gateway** (router).  
- Laptop biết IP gateway, nhưng cần MAC của gateway → dùng ARP:
  - ARP request broadcast: “Ai có IP `<gateway>`?”.  
  - Gateway trả ARP reply: “IP `<gateway>` là MAC `<mac_gateway>`”.  
- Laptop đóng gói frame Ethernet:
  - MAC nguồn = MAC của laptop.  
  - MAC đích = MAC của gateway.  
- Switch / AP dùng MAC table để chuyển frame tới đúng cổng của gateway.

Lệnh kiểm tra:

```bash
arp -a          # xem ARP cache (IP <-> MAC)
ping <gateway>  # kiểm tra có reach được gateway không
```

#### 4.2. Ví dụ LAN công ty

- Laptop: `10.20.1.50/24`, gateway `10.20.1.1`.  
- Switch nối laptop với router.  
- Frame sẽ đi: Laptop → Switch → Router.

Nếu không ping được gateway:

- Có thể lỗi WiFi, cáp, cấu hình IP, VLAN, port switch, ARP…  
- Với Data Engineer, chỉ cần nhận ra: vấn đề nằm **trong LAN**, chưa lên tới ISP hay cloud.


---

### 5. Bước 3 – Router, NAT và ISP

#### 5.1. Router & NAT trên modem

Trong mạng gia đình / nhiều văn phòng nhỏ:

- Modem/router thường:
  - Là **gateway IP** cho LAN (`192.168.x.1`).  
  - Làm **NAT**: chuyển IP private (192.168.x.x, 10.x.x.x…) của LAN thành một IP public của nhà mạng.

Luồng:

- Laptop gửi gói đến gateway.  
- Router thay IP nguồn (private) bằng IP public, gán port NAT, gửi ra ISP.  
- Khi server trả lời, router nhận về, map ngược port NAT → IP private → gửi frame lại cho laptop.

> [!NOTE]
> NAT giúp nhiều thiết bị nội bộ dùng chung một IP public. Trong cloud, có NAT Gateway làm nhiệm vụ tương tự cho private subnet.

#### 5.2. ISP và đường đi trên Internet

Sau khi ra khỏi router:

- Gói tin đi qua nhiều router của ISP, các mạng trung gian, rồi tới mạng của cloud provider.  
- Mỗi router chỉ nhìn IP nguồn/đích và bảng định tuyến để quyết định hop tiếp theo.

Lệnh kiểm tra:

```bash
ping 8.8.8.8             # kiểm tra kết nối tới một IP ngoài Internet
traceroute 8.8.8.8       # Linux/macOS
tracert 8.8.8.8          # Windows
```

`traceroute`/`tracert` cho thấy các hop (router) mà gói tin đi qua. Nếu bị kẹt ở hop nào, có thể đoán vấn đề ở đoạn đó (ISP, quốc tế, peering…).


---

### 6. Bước 4 – Từ Internet vào cloud / data center

#### 6.1. Edge router, load balancer, VPC

Ở phía cloud (ví dụ GCP, AWS, Azure):

```mermaid
flowchart LR
    A[Internet] --> B[Edge Router / Firewall]
    B --> C[Load Balancer / API Gateway]
    C --> D[VPC Router]
    D --> E[Private Subnet<br/>(App / DB / DW / Kafka...)]
```

- **Edge Router / Firewall**:
  - Điểm vào đầu tiên từ Internet, kiểm soát firewall, DDoS, v.v.  
- **Load Balancer / API Gateway**:
  - Nhận kết nối từ client, phân phối tới backend, có thể terminate TLS.  
- **VPC Router (route table)**:
  - Định tuyến gói tới subnet/private IP của service.  
- **Service trong VPC**:
  - Web service, API, database, warehouse, cluster Kafka/Spark/… với IP private.

#### 6.2. Public vs Private endpoint

- **Public endpoint**:
  - Có IP public; client trên Internet gọi trực tiếp (qua LB).  
- **Private endpoint / Private Link**:
  - Chỉ accessible từ VPC / mạng được phép (VPN, peering), không mở trực tiếp ra Internet.

Data Engineer thường gặp:

- Public console hoặc API cho BigQuery, Snowflake, Databricks.  
- Private endpoint cho Cloud SQL / RDS / internal API, chỉ truy cập được qua VPN/bastion hoặc VPC peering.


---

### 7. Ví dụ: từ laptop tới BigQuery / Snowflake console

Sơ đồ tổng quát:

```mermaid
flowchart LR
    L[Laptop] --> W[WiFi Router / LAN]
    W --> M[Modem Router / NAT]
    M --> I[ISP / Internet]
    I --> CE[Cloud Edge Router]
    CE --> LB[Cloud Load Balancer]
    LB --> SV[Service BigQuery / Snowflake<br/>(trong VPC)]
```

Luồng logic:

1. Browser mở URL console.  
2. Laptop:
   - DNS → IP của endpoint.  
   - TCP + TLS → port 443.  
3. Gói đi qua LAN → router/NAT → ISP → Internet backbone.  
4. Tới edge cloud:
   - Firewall/edge router → LB/API gateway → service trong VPC.  

> [!WARNING]
> Mọi lỗi từ: “không mở được trang”, “SSL error”, “console lúc được lúc không”… đều nằm đâu đó trên đường này. Khi debug, cần tách rời:
> - Laptop / LAN.  
> - ISP / đường quốc tế.  
> - Phía cloud / service.


---

### 8. Các điểm dễ phát sinh lỗi trên đường từ laptop đến Internet

1. **Laptop / hệ điều hành**
   - IP sai / xung đột, gateway sai.  
   - DNS cấu hình sai hoặc bị override.  
   - Local firewall / VPN client chặn.

2. **WiFi / Switch nội bộ**
   - Mất kết nối, tín hiệu yếu.  
   - VLAN sai, port bị shutdown hoặc filter.

3. **Router / NAT / modem**
   - Không phải default gateway.  
   - NAT / firewall chặn outbound tới port/địa chỉ nhất định.  
   - Bị treo, quá tải, lỗi cấu hình.

4. **ISP / đường Internet**
   - Đứt cáp, nghẽn tuyến quốc tế, lỗi peering.  
   - Một số IP / prefix bị filter.

5. **Phía cloud / data center**
   - Firewall / security group / NACL chặn IP hoặc port.  
   - Sai route table, sai mapping private–public endpoint.  
   - Load balancer / API gateway cấu hình sai, healthcheck fail.

6. **Layer ứng dụng**
   - Service trả lỗi 4xx/5xx, rate limit, auth/token sai.  
   - TLS cấu hình sai (hostname, cert, cipher…).

---

### 9. Sổ tay lệnh debug theo từng đoạn

**Trên laptop:**

```bash
# Kiểm tra IP, gateway, DNS
ipconfig /all                # Windows
ip addr show; ip route       # Linux / macOS
cat /etc/resolv.conf         # DNS trên Linux
```

**Kiểm tra LAN & gateway:**

```bash
ping <gateway-ip>            # ví dụ: ping 192.168.1.1
arp -a                       # xem ARP cache
```

**Kiểm tra kết nối Internet chung:**

```bash
ping 8.8.8.8                 # kiểm tra đến Google DNS
traceroute 8.8.8.8           # hoặc 'tracert 8.8.8.8' trên Windows
```

**Kiểm tra DNS & service cụ thể:**

```bash
dig <hostname>               # hoặc 'nslookup <hostname>'
ping <hostname>
traceroute <hostname>        # hoặc 'tracert'
curl -v https://<hostname>   # xem chi tiết TLS + HTTP
```

> [!TIP]
> Khi có sự cố, nên “đi từ gần đến xa”:  
> 1) Laptop → gateway.  
> 2) Gateway → Internet (ping 8.8.8.8).  
> 3) Internet → endpoint cụ thể (ping / traceroute / curl).  
> Tới đâu hỏng thì dừng lại và tập trung vào đoạn đó.


---

### 10. Tóm tắt

- Đường đi từ laptop tới một dịch vụ trên cloud luôn gồm nhiều đoạn: **LAN → router/NAT → ISP → Internet → edge cloud → LB → VPC/service**.  
- Ở mỗi đoạn có thiết bị, giao thức và cấu hình riêng (IP, MAC, ARP, route, NAT, firewall, TLS…), bất kỳ đoạn nào hỏng đều làm “app không kết nối được”.  
- Với người làm Data/Analytics, hiểu được đường đi này:
  - Giúp phân biệt lỗi thuộc phần mình hay thuộc hạ tầng.  
  - Giúp nói chuyện hiệu quả với team mạng / cloud / SRE.  
  - Là nền tảng bắt buộc trước khi đi vào các phase tiếp theo về DNS, TCP, HTTP, Cloud Networking, Data Platform Networking.