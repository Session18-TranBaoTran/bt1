# Session16 – Ex01: GreenLogistics – DNS & HTTP Status Code


## 1. Bối cảnh

Nam là thực tập sinh tại GreenLogistics, cần truy cập trang quản trị `phong-dieu-phoi.greenlog.vn` để cập nhật lộ trình xe tải. Trình duyệt liên tục báo lỗi `DNS_PROBE_FINISHED_NXDOMAIN`. Nam cho rằng máy chủ công ty bị sập nguồn.

**Thực tế:** Nam đang gõ nhầm thành `phong-dieu-phoi.grenlog.vn` (thiếu một chữ `e`).

---

## 2. Phần 1 – Phân tích

### 2.1. Input / Output

| Thành phần | Nội dung |
|---|---|
| **Input** | Tên miền: `phong-dieu-phoi.greenlog.vn` |
| **Process** | DNS Lookup (phân giải tên miền) |
| **Output** | Địa chỉ IP của máy chủ (ví dụ `203.0.113.25`) hoặc lỗi **NXDOMAIN** nếu tên miền không tồn tại |

DNS giống như **cuốn danh bạ Internet**: con người nhớ *tên*, máy tính cần *số (IP)*.

### 2.2. Tiến trình DNS Lookup

1. **Kiểm tra cache cục bộ:** trình duyệt → hệ điều hành → file `hosts`. Nếu đã có IP thì dùng ngay.
2. **Hỏi Recursive Resolver:** DNS của nhà mạng (ISP) hoặc `8.8.8.8`, `1.1.1.1`. Resolver đi tìm câu trả lời thay cho người dùng.
3. **Hỏi Root Server (`.`):** root không biết IP, chỉ chỉ đường: "hãy hỏi máy chủ quản lý `.vn`".
4. **Hỏi TLD Server (`.vn`):** trả lời: "hãy hỏi Authoritative Server của `greenlog.vn`".
5. **Hỏi Authoritative Name Server của `greenlog.vn`:** nơi lưu bản ghi chính thức (A/AAAA) và trả về IP.
6. **Trả kết quả:** Resolver lưu cache (theo TTL) và gửi IP cho trình duyệt. Trình duyệt dùng IP để kết nối tới Web Server.

### 2.3. Sơ đồ luồng dữ liệu

```text
 Nam (Browser)
     │ 1. Gõ phong-dieu-phoi.greenlog.vn
     ▼
 [Cache trình duyệt / OS / hosts] ── có? ──► dùng IP luôn
     │ không
     ▼
 [Recursive Resolver (ISP / 8.8.8.8)]
     │ 2. hỏi          ▲ 6. trả IP (hoặc NXDOMAIN)
     ▼                 │
 [Root Server "."] → [TLD ".vn"] → [Authoritative NS của greenlog.vn]
        "hỏi .vn"       "hỏi greenlog.vn"      "IP là 203.0.113.25"
     │
     ▼
 Browser nhận IP ──► kết nối tới Web Server ──► nhận HTTP Status Code
```

![Sơ đồ DNS Lookup](images/04-dns-flow.png)

### 2.4. Hậu quả của việc gõ sai tên miền (`grenlog.vn`)

Trong "cuốn danh bạ Internet", `grenlog.vn` là **một tên hoàn toàn khác** với `greenlog.vn`. DNS so khớp chính xác từng ký tự, không "đoán ý" người dùng.

- Authoritative Server không có bản ghi cho `grenlog.vn` và trả về **NXDOMAIN** (Non-Existent Domain).
- Trình duyệt **không có IP** nên không thể mở kết nối. Yêu cầu **chưa hề tới máy chủ web**.
- Rủi ro bảo mật: nếu ai đó đăng ký `grenlog.vn` và dựng trang giả thì đây là tấn công **typosquatting**.

**Kết luận:** khẳng định "máy chủ sập nguồn" của Nam là **sai**. Lỗi nằm ở phía **Client** (gõ sai tên miền), máy chủ công ty có thể vẫn hoạt động bình thường.

---

## 3. Chẩn đoán lỗi: DNS và lỗi 500

| Tiêu chí | `DNS_PROBE_FINISHED_NXDOMAIN` | `500 Internal Server Error` |
|---|---|---|
| **Giai đoạn xảy ra** | Phân giải tên (trước khi kết nối) | Xử lý ứng dụng (sau khi đã kết nối) |
| **Bản chất** | Tên miền không có trong DNS, không ra IP | Server nhận request nhưng code/cấu hình bị lỗi khi xử lý |
| **Request đã tới server chưa?** | **Chưa** | **Rồi** |
| **Có HTTP Status Code không?** | **Không** | **Có** (500) |
| **Phía gây lỗi** | Thường là **Client** (gõ sai, cache DNS, cấu hình DNS) | **Server** |
| **Cách xử lý** | Kiểm tra chính tả, `nslookup`, xóa cache DNS, đổi DNS | Xem log server, báo đội backend/DevOps |

![Request dừng ở đâu](images/06-so-sanh-loi.png)

> **Tóm lại:** 500 nghĩa là server "còn sống" nhưng xử lý lỗi. NXDOMAIN nghĩa là **chưa tìm thấy đường tới server**.

---

## 4. Phần 2 – Triển khai: quy trình kiểm tra HTTP Status Code

**Bước 1. Kiểm tra chính tả** tên miền trên thanh địa chỉ: `greenlog.vn` hay `grenlog.vn`?

**Bước 2. Kiểm tra DNS bằng terminal:**

```bash
nslookup phong-dieu-phoi.greenlog.vn
nslookup phong-dieu-phoi.grenlog.vn
```

- Tên đúng thì trả về IP. Tên sai thì báo `Non-existent domain` (NXDOMAIN).
- Nếu tên đúng mà vẫn lỗi, xóa cache DNS (`ipconfig /flushdns` trên Windows) hoặc đổi DNS sang `8.8.8.8`.

**Bước 3. Mở DevTools (F12) → tab Network:**

1. Nhấn **F12**, chọn tab **Network**, tick **Preserve log**.
2. Truy cập `https://phong-dieu-phoi.greenlog.vn`.
3. Quan sát dòng request đầu tiên, cột **Status**:
   - Không có dòng nào, hoặc `(failed)` / `ERR_NAME_NOT_RESOLVED`: yêu cầu **chưa chạm tới server** (lỗi DNS).
   - Có mã trạng thái: yêu cầu **đã chạm tới server**, đọc mã theo bảng bên dưới.

**Bước 4. (Tùy chọn) Kiểm tra bằng lệnh:**

```bash
curl -I https://phong-dieu-phoi.greenlog.vn
```

Dòng đầu (ví dụ `HTTP/2 200`) cho biết mã trạng thái.

### Bảng đọc mã trạng thái

| Nhóm | Ý nghĩa | Ví dụ | Kết luận |
|---|---|---|---|
| 2xx | Thành công | 200 OK | Server hoạt động bình thường |
| 3xx | Chuyển hướng | 301, 302 | Server phản hồi, request bị redirect |
| 4xx | Lỗi phía Client | 404, 403, 401 | Đã tới server, request/đường dẫn/quyền sai |
| 5xx | Lỗi phía Server | 500, 502, 503 | Đã tới server, server xử lý lỗi |

### Sơ đồ quyết định

```text
Gõ URL → Có phân giải ra IP không?
          ├─ Không → Lỗi DNS (NXDOMAIN) → Client sửa tên miền / DNS
          └─ Có → Có nhận Status Code không?
                    ├─ Không (timeout) → Lỗi mạng / firewall / server tắt
                    └─ Có → 2xx ổn | 4xx lỗi Client | 5xx lỗi Server
```

![Cây quyết định chẩn đoán](images/05-decision-tree.png)

### Ảnh minh họa (F12)

| Trường hợp | Ảnh |
|---|---|
| Truy cập `grenlog.vn` (sai) → `ERR_NAME_NOT_RESOLVED` | ![Lỗi DNS](images/01-dns-error.png) |
| Truy cập đúng tên miền → có Status Code | ![Status Code](images/02-status-code.png) |
| Kết quả `nslookup` | ![nslookup](images/03-nslookup.png) |

---

## 5. Phần 3 – Kiểm thử: sửa đúng tên miền nhưng vẫn nhận 404

**Phân tích:**

- Mã **404 Not Found** chứng tỏ DNS **đã thành công** (đã có IP) và request **đã tới máy chủ**. Giả thuyết "server sập nguồn" của Nam hoàn toàn bị bác bỏ.
- Server đang chạy và phản hồi, nhưng **không tìm thấy tài nguyên** tại đường dẫn được yêu cầu.

**Các nguyên nhân có thể:**

1. **Sai đường dẫn (path):** trang quản trị nằm ở `/admin` hoặc `/dieu-phoi/lo-trinh` chứ không phải `/`, hoặc Nam gõ sai/thiếu path.
2. **Trang/file đã bị xóa, đổi tên hoặc di chuyển** mà chưa cấu hình redirect.
3. **Triển khai (deploy) sai:** thiếu file, sai thư mục gốc (document root), cấu hình virtual host/reverse proxy trỏ nhầm.
4. **Lỗi routing của ứng dụng:** route chưa được khai báo.
5. **Tên miền trỏ đúng IP nhưng website chưa được cấu hình** cho tên miền đó.

**Hướng xử lý:** kiểm tra lại URL đầy đủ, hỏi quản trị viên đường dẫn đúng, xem access/error log của server, rà soát cấu hình routing và deploy.

**Kết luận:** lỗi 404 thuộc nhóm **4xx (phía yêu cầu/đường dẫn)**, khác bản chất với NXDOMAIN (chưa tìm ra server) và 500 (server xử lý hỏng).

---

## 6. Kết luận chung

| Hiện tượng | Nguyên nhân | Phía lỗi |
|---|---|---|
| `DNS_PROBE_FINISHED_NXDOMAIN` | Gõ sai tên miền (`grenlog.vn`) | Client |
| `404 Not Found` (sau khi gõ đúng) | Sai đường dẫn / tài nguyên không tồn tại / cấu hình sai | Client hoặc cấu hình triển khai |
| `500 Internal Server Error` | Lỗi xử lý bên trong server | Server |

Nam **không thể** kết luận "server sập" chỉ dựa vào lỗi NXDOMAIN. Cần kiểm tra theo thứ tự: **chính tả → DNS (`nslookup`) → Status Code (F12 Network)**.
