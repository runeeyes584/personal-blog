---
title: "Nghệ Thuật 'Dọn Rác' (Java Garbage Collection - GC) Tối Ưu"
meta_title: ""
description: "this is meta description"
date: 2025-05-04T05:00:00Z
image: "/images/gallery/03.jpg"
categories: ["Technology", "Data", "Java"]
author: "Lê Anh Tiến"
tags: ["java", "technology"]
draft: false
---

Nghệ Thuật "Dọn Rác" (Java Garbage Collection - GC) Tối Ưu: GC mà chạy "dở" là app "đứng hình" liền! So sánh các "chiến binh" dọn rác như G1GC, ZGC, Shenandoah. Hướng dẫn cách "tuning" (tinh chỉnh) JVM và GC để ứng dụng Java chạy mượt "không tì vết".

---

# Nghệ Thuật “Dọn Rác” (Java GC) Tối Ưu
**Nếu GC “dở”, ứng dụng sẽ “đứng hình”. Chọn đúng collector, log đúng số liệu, và tinh chỉnh có phương pháp để hệ thống chạy mượt.**

## 1) Bản chất vấn đề: Vì sao GC ảnh hưởng đến độ mượt?
- Java cấp phát đối tượng rất nhanh (bump-the-pointer) → rác sinh ra liên tục.
- Khi heap đầy, GC cần thu hồi và **đôi khi dừng ứng dụng (Stop-The-World – STW)** ở vài pha bắt buộc.
- Mục tiêu tối ưu:
  - **Latency**: giảm độ dài & độ lệch (tail) của pause.
  - **Throughput**: tối đa thời gian CPU chạy ứng dụng (ít chi phí GC).
  - **Footprint**: bộ nhớ dùng hợp lý, tránh “phình to” không cần thiết.

> Mỗi collector tối ưu một “tam giác” khác nhau: **độ trễ thấp**, **thông lượng cao**, **độ phức tạp quản trị**.

---

## 2) Toàn cảnh các “chiến binh” GC hiện đại

### 2.1 G1GC (Garbage-First)
- **Mặc định** ở nhiều JDK hiện đại cho heap trung bình–lớn.
- Heap chia thành **regions**; thu gom theo **vùng có nhiều rác trước**; có **concurrent marking**, **cân bằng** giữa throughput và latency.
- **Tối ưu**: hệ thống dịch vụ điển hình, latency mục tiêu “hàng chục ms”, heap từ vài GB đến hàng chục GB.
- **Điểm cần lưu ý**:
  - **Humongous objects** (đối tượng > 50% kích thước region) gây áp lực lên heap; cân nhắc **tăng `G1HeapRegionSize`** hoặc tránh tạo chuỗi byte cực lớn.
  - Mục tiêu `MaxGCPauseMillis` **quá thấp** có thể làm **tăng chi phí nền**, giảm throughput.

### 2.2 ZGC (Z Garbage Collector)
- Nhắm tới **độ trễ cực thấp** (mục tiêu pause ~ vài ms) **trên heap rất lớn** (từ hàng chục đến hàng trăm GB+).
- **Concurrent compaction** thực thụ, **colored pointers / load barriers** giúp di chuyển đối tượng gần như không cần STW dài.
- **Thích hợp**: dịch vụ **latency-critical**, heap lớn, workload I/O-bound nhiều request đồng thời.
- **Lưu ý**:
  - Chi phí barrier và metadata; nên **đo đạc** vì không phải workload nào cũng cần ZGC.
  - ZGC có cơ chế **uncommit** bộ nhớ nhàn rỗi; chú ý các tham số trì hoãn uncommit để tránh “sóng” cấp-phát lại.

### 2.3 Shenandoah
- Mục tiêu **low-pause** tương tự ZGC, với cơ chế **concurrent compaction** và **brooks forwarding pointer**.
- **Thích hợp**: latency thấp ổn định, heap lớn; thường dùng trong môi trường yêu cầu pause thấp nhưng không cần mọi đặc trưng của ZGC.
- **Lưu ý**:
  - Có nhiều **heuristics** vận hành; chọn đúng heuristic giúp ổn định hơn theo hình dạng workload.

> Tóm tắt chọn nhanh  
> - **Ưu tiên throughput, cấu hình đơn giản** → **G1GC**.  
> - **Ưu tiên latency rất thấp, heap lớn** → **ZGC** (hoặc **Shenandoah** nếu phù hợp môi trường/tooling).  
> - **Batch CPU-bound** (không quan trọng pause) → Parallel GC cũng là một lựa chọn throughput cao, nhưng bài này tập trung G1/ZGC/Shenandoah.

---

## 3) Tư duy “tuning” đúng cách: Quy trình 6 bước

### Bước 0 — Đặt SLO
- Ví dụ: **p99 latency < 100 ms**, **p50 < 20 ms**, **thruput ≥ N req/s**, **heap ≤ 8 GB**.

### Bước 1 — Bật log GC và đo
```bash
# JDK 11+ unified logging
-Xlog:gc*,safepoint:file=gc.log:time,uptime,level,tags
````

Phân tích: count, thời lượng pause (min/avg/p95/p99), allocation rate, promotion, bộ sưu tập concurrent.

### Bước 2 — Xác định pattern

* Pause dài xảy ra ở **Young** hay **Mixed/Old**?
* Promotion thất bại, to_compact quá nhiều?
* Có **Humongous** trong G1? ZGC/Shenandoah có “evacuation failure”?

### Bước 3 — Chọn GC & cấu hình nền

* G1 (mặc định), hoặc bật ZGC/Shenandoah (xem §4).
* Đặt **heap đủ lớn** để giảm tần suất GC (ít nhất gấp 2–3 lần dữ liệu sống).
* Bật **container awareness** (mặc định JDK mới) và dùng %

```bash
-XX:InitialRAMPercentage=25 -XX:MaxRAMPercentage=75
```

### Bước 4 — Tinh chỉnh theo mục tiêu

* Nếu **pause cao**: nhắm mục tiêu pause, mở rộng heap, tăng thread GC, giảm humongous.
* Nếu **throughput thấp**: nới `MaxGCPauseMillis`, giảm hoạt động concurrent, hoặc cân đối heap.

### Bước 5 — Kiểm tra hồi quy & tải

* Dùng **JFR**, **load test** (soak test), kiểm tra **p99/p999**, tail latency dưới áp lực.

### Bước 6 — Duy trì & giám sát

* Theo dõi GC log/JFR liên tục; cảnh báo khi pause vượt SLO hoặc allocation rate thay đổi lớn.

---

## 4) Cấu hình căn bản theo collector

### 4.1 G1GC

Bật (nếu chưa là mặc định):

```bash
-XX:+UseG1GC
```

Các tham số hay dùng:

```bash
-XX:MaxGCPauseMillis=100             # mục tiêu pause (ms)
-XX:InitiatingHeapOccupancyPercent=45 # bắt đầu marking khi old occupancy đạt %
-XX:G1HeapRegionSize=4m               # 1–32m; cân nhắc tăng nếu nhiều humongous
-XX:G1ReservePercent=15               # vùng dự phòng tránh promotion failure
-XX:ConcGCThreads=4                   # số thread concurrent
-XX:ParallelGCThreads=8               # thread GC song song
-XX:+UseStringDeduplication           # khử trùng lặp chuỗi (cân nhắc đo đạc)
```

Gợi ý:

* Nếu **young GC quá thường xuyên**, tăng heap hoặc nới mục tiêu pause.
* Nếu **Mixed GC kéo dài**, giảm `InitiatingHeapOccupancyPercent` để marking sớm hơn.
* Tránh tạo **đối tượng rất lớn** liên tục; cân nhắc **chunking**.

### 4.2 ZGC

Bật:

```bash
-XX:+UseZGC
```

Một số tham số:

```bash
-XX:SoftMaxHeapSize=8g      # "mềm" để ZGC điều tiết footprint
-XX:ZUncommitDelay=300s     # bao lâu mới nhả lại bộ nhớ nhàn rỗi
-XX:ConcGCThreads=4         # số thread concurrent
-XX:ParallelGCThreads=8     # thread song song
# Nếu dùng huge pages/NUMA, kiểm tra môi trường trước khi bật
```

Gợi ý:

* ZGC phù hợp **heap lớn** và **latency rất thấp**. Đừng “ép” ZGC cho workload nhỏ nếu G1 đã đạt SLO.
* Đo **chi phí barrier** và **allocation rate**; tối ưu mô hình cấp phát trước khi tăng heap vô hạn.

### 4.3 Shenandoah

Bật:

```bash
-XX:+UseShenandoahGC
```

Tham số thường dùng:

```bash
-XX:ShenandoahGCHeuristics=adaptive   # hoặc compact/static/aggressive (đo rồi chọn)
-XX:ShenandoahUncommitDelay=300s
-XX:ConcGCThreads=4
-XX:ParallelGCThreads=8
# String dedup: kiểm tra JDK build của bạn, bật nếu phù hợp
```

Gợi ý:

* Chọn **heuristics** phù hợp hình dạng workload (allocation burst, steady-state).
* Theo dõi **evacuation** và **degeneration** trong log để tránh phase fallback.

---

## 5) Tối ưu heap & vùng nhớ liên quan

### 5.1 Kích cỡ heap

```bash
-Xms4g -Xmx8g
```

* **Xms gần bằng Xmx** giúp tránh “đàn hồi” heap và phân mảnh ban đầu.
* Trong container, ưu tiên **% RAM** thay vì cố định GB.

### 5.2 Vùng non-heap

* **Metaspace**: theo dõi `-XX:MaxMetaspaceSize` nếu lớp nạp/huỷ thường xuyên.
* **Direct buffers**: kiểm soát qua `-XX:MaxDirectMemorySize` khi dùng NIO/Netty.
* Theo dõi **native memory**: `jcmd VM.native_memory summary`.

### 5.3 Kiểm soát cấp phát

* Giảm **boxing**, dùng **primitive collections** khi phù hợp.
* Tái sử dụng buffer/pool cẩn trọng (tránh giữ tham chiếu không cần → rò rỉ).
* Tránh build chuỗi lớn liên tục; dùng streaming/chunk.

---

## 6) Quy trình chẩn đoán sự cố GC

1. **Bật log GC/JFR**:

   ```bash
   -Xlog:gc*,safepoint:file=gc.log:time,level,tags
   -XX:StartFlightRecording=filename=app.jfr,dumponexit=true,settings=profile
   ```
2. **Kiểm tra pause spike**: liên hệ **safepoint** bất thường, lock contention.
3. **Xem allocation rate & live set**: nếu **live set** gần Xmx, hãy tăng heap hoặc giảm rò rỉ.
4. **Phân tích heap dump** khi nghi ngờ rò rỉ:

   ```bash
   jcmd <pid> GC.heap_dump /path/heap.hprof
   ```
5. **So sánh cấu hình**: thử nghiệm có kiểm soát với **1 thay đổi mỗi lần**, đánh giá bằng **p95/p99**.

---

## 7) Tuning theo kịch bản thực tế

### 7.1 Dịch vụ web latency-sensitive (heap 8–32 GB)

* Khởi đầu với G1GC:

  ```bash
  -XX:+UseG1GC
  -Xms16g -Xmx16g
  -XX:MaxGCPauseMillis=100
  -XX:InitiatingHeapOccupancyPercent=40
  -XX:G1ReservePercent=20
  -Xlog:gc*
  ```
* Nếu vẫn có **p99 > 150 ms** dù đã tối ưu allocation, cân nhắc **ZGC**:

  ```bash
  -XX:+UseZGC
  -Xms16g -Xmx16g
  -Xlog:gc*
  ```

### 7.2 Batch/ETL CPU-bound, không nhạy pause

* Ưu tiên **throughput** hơn latency:

  * G1GC với `MaxGCPauseMillis` “nới lỏng” hoặc **Parallel GC**.
* Quan trọng là **thời gian hoàn thành tổng** và **chi phí CPU**.

### 7.3 Ứng dụng microservice containerized

```bash
-XX:InitialRAMPercentage=25
-XX:MaxRAMPercentage=70
-XX:+AlwaysPreTouch                 # ổn định phân trang nếu đủ tài nguyên
-XX:+UseContainerSupport            # mặc định mở ở JDK mới
```

---

## 8) Những “antipattern” phổ biến

* **Đặt `MaxGCPauseMillis` quá thấp** → collector gồng mình, throughput giảm, tốn CPU GC.
* **Heap quá nhỏ** → GC dồn dập, promotion thất bại, pause tăng.
* **Tạo đối tượng lớn/humongous liên tục (G1)** → phân mảnh/vỡ kế hoạch region.
* **Không log/không đo** → tinh chỉnh mù mờ, khó tái hiện.
* **Giữ tham chiếu lâu** (cache không TTL/size) → live set phình, GC bất lực.

---

## 9) Bảng so sánh nhanh

| Tiêu chí               | G1GC                            | ZGC                                 | Shenandoah                          |
| ---------------------- | ------------------------------- | ----------------------------------- | ----------------------------------- |
| Mục tiêu chính         | Cân bằng latency/throughput     | Latency cực thấp, heap rất lớn      | Latency thấp, concurrent compaction |
| Kiến trúc              | Region-based, mixed collections | Concurrent compaction, colored ptrs | Concurrent compaction, forwarding   |
| Phạm vi heap điển hình | GB → chục GB                    | Chục GB → trăm GB+                  | GB → trăm GB                        |
| Độ phức tạp tuning     | Vừa                             | Thấp–vừa (đo là chính)              | Vừa (chọn heuristic phù hợp)        |
| Trường hợp nên chọn    | Dịch vụ web phổ dụng            | Latency-critical, heap lớn          | Latency thấp ổn định                |

---

## 10) Kết luận & khuyến nghị

* **Bắt đầu với G1GC**: đủ tốt cho hầu hết dịch vụ; dễ đạt SLO nếu heap hợp lý và allocation kiểm soát.
* **Nhu cầu latency cực thấp/heap rất lớn**: chuyển sang **ZGC** (hoặc **Shenandoah** nếu phù hợp hệ sinh thái).
* **Tuning là khoa học thực nghiệm**: luôn **đo đạc** (GC log, JFR), **đổi một biến mỗi lần**, so sánh theo **p95/p99** và throughput.
* **Kỹ thuật nền**: giảm cấp phát vô ích, tránh humongous, cài đặt heap đủ rộng, giám sát liên tục.

> GC tốt là kết quả của **chọn collector đúng + heap phù hợp + kỷ luật đo đạc**. Khi làm đúng, ứng dụng sẽ “mượt không tì vết”.

---