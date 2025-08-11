# Tự động xóa ghi âm và CDR từ danh sách Google Sheet

## 1. Giới thiệu

Hệ thống này được thiết kế để **tự động xóa ghi âm** (recordings) và/hoặc **CDR** (Call Detail Records) của các máy ảo (VM) dựa trên danh sách cấu hình lưu trên **Google Sheets** hoặc file CSV.

Điểm nâng cấp mới:

* Có thể chọn **hành động xóa**: chỉ recordings, chỉ CDR, cả hai, hoặc bỏ qua.
* Hỗ trợ thông tin `CONTEXT_TYPE` và `AUTH` riêng cho từng tenant.
* Có chế độ **`--dry-run`** để kiểm tra trước khi xóa.

---

## 2. Lợi ích

* Giảm thao tác thủ công.
* Giảm rủi ro sai sót khi vận hành.
* Tiết kiệm thời gian và tài nguyên lưu trữ.
* Cho phép linh hoạt tùy chọn hành động xóa.

---

## 3. Lưu ý về dữ liệu CDR

Trong hệ thống Autocall/FreeSWITCH:

* **CDR** được lưu ở:

  1. **Database (DB)** — thường là PostgreSQL/MySQL.
  2. **ElasticSearch (ES)** — phục vụ log & tìm kiếm nhanh.
* Script sẽ xóa:

  * **Recordings** trong thư mục `/usr/local/freeswitch/recordings/<tenant>/archive/`.
  * **CDR trên ES** qua API `_delete_by_query`.
  * **CDR trong DB** qua lệnh `DELETE`.

---

## 4. Luồng hoạt động

![Luồng hoạt động](cron_clear_recordings_cdr_flow.png)

**Mô tả:**

1. Crontab chạy script vào giờ định sẵn.
2. Script tải danh sách tenant từ Google Sheets (xuất CSV).
3. Đọc từng dòng: tenant, thời gian xóa, thông tin DB, ES, và hành động xóa (`Action`).
4. Thực hiện xóa recordings / CDR / cả hai theo tùy chọn.
5. Ghi log kết quả.

---

## 5. Cấu trúc Google Sheet / CSV

**Ví dụ:**

| STT | Tenant Name | Routing UUID                                  | From Datetime       | To Datetime         | Alias Index      | API URL                                          | CONTEXT\_TYPE    | AUTH                       | DB Host    | DB Port | DB Name   | DB User   | DB Pass | Action     | Note                |
| --- | ----------- | --------------------------------------------- | ------------------- | ------------------- | ---------------- | ------------------------------------------------ | ---------------- | -------------------------- | ---------- | ------- | --------- | --------- | ------- | ---------- | ------------------- |
| 1   | tenant1@tenant.com     | tenant1\_a7843225-8806-4e3f-b183-9df24fe6b68f | 2024-07-01 00:00:00 | 2024-08-31 23:59:59 | tenant1\_a7843225-8806-4e3f-b183-9df24fe6b68f | [http://10.10.10.5:9200](http://10.10.10.5:9200) | application/json | Basic YWRtaW46cGFzc3dvcmQ= | 10.10.10.2 | 5432    | fusionpbx | fusionpbx | pass123 | both       | Xóa dữ liệu 2 tháng |
| 2   | tenant2@tenant.com      | tenant2\_b1523421-8806-4e3f-b183-9df24fe6b68f | 2024-06-01 00:00:00 | 2024-06-30 23:59:59 | tenant2\_a7843225-8806-4e3f-b183-9df24fe6b68f | [http://10.10.10.5:9200](http://10.10.10.5:9200) | application/json | Bearer abc123              | 10.10.10.3 | 3306    | autocall  | autocall  | pass456 | recordings | Chỉ xóa file ghi âm |


**Giải thích các cột:**

* `STT`
  Số thứ tự trong danh sách, dùng để theo dõi, không ảnh hưởng đến script.

* `Tenant Name`
  Tên tenant trên hệ thống, cũng là tên thư mục chứa recordings: `/usr/local/freeswitch/recordings/<Tenant Name>/archive/`.

* `Routing UUID`
  Giá trị `_routing` dùng để lọc khi xóa CDR trên ElasticSearch.

* `From Datetime`
  Thời điểm bắt đầu của khoảng thời gian cần xóa dữ liệu, định dạng `YYYY-MM-DD HH:MM:SS`.

* `To Datetime`
  Thời điểm kết thúc của khoảng thời gian cần xóa dữ liệu, định dạng `YYYY-MM-DD HH:MM:SS`.

* `Index UUID`
  Tên index ElasticSearch chứa dữ liệu CDR cần xóa.

* `API URL`
  Địa chỉ API ElasticSearch, ví dụ: `http://10.10.10.5:9200`.

* `CONTEXT_TYPE`
  Content-Type header khi gọi API ElasticSearch, ví dụ: `application/json`.

* `AUTH`
  Thông tin Authorization header khi gọi API ElasticSearch, ví dụ: `Basic YWRtaW46cGFzc3dvcmQ=` hoặc `Bearer <token>`. Có thể để trống nếu không cần.

* `DB Host`
  Địa chỉ IP hoặc hostname của database chứa CDR.

* `DB Port`
  Cổng kết nối đến database, ví dụ: `5432` cho PostgreSQL, `3306` cho MySQL.

* `DB Name`
  Tên database chứa bảng CDR (thường bảng `v_xml_cdr`).

* `DB User`
  Tài khoản database có quyền xóa CDR.

* `DB Pass`
  Mật khẩu của tài khoản database.

* `Action`
  Hành động muốn thực hiện với tenant:

  * `recordings` → chỉ xóa file ghi âm.
  * `cdr` → chỉ xóa CDR (ElasticSearch + DB).
  * `both` → xóa cả hai (mặc định nếu để trống).
  * `none` → bỏ qua tenant này.

* `Note`
  Ghi chú cho quản trị viên, ví dụ: “Xóa dữ liệu 2 tháng”.

---

## 6. Cài đặt script

**File script**: `auto_clear_records_cdr.sh`
Quyền: `chmod +x auto_clear_records_cdr.sh`

**Chạy thử (không xóa thật):**

```bash
/bin/bash auto_clear_records_cdr.sh --dry-run
```

**Chạy thật:**

```bash
/bin/bash auto_clear_records_cdr.sh
```

---

## 7. Cài đặt crontab

Chạy mỗi ngày lúc 03:00 sáng:

```bash
0 3 * * * /bin/bash /home/user/scripts/auto_clear_records_cdr.sh
```

---

## 8. Yêu cầu hệ thống

* Máy chủ cài:

  * `curl`, `find`, `psql` (PostgreSQL), `mysql` (MySQL).
* Quyền đọc/ghi thư mục recordings.
* API ElasticSearch `_delete_by_query` hoạt động.
* Kết nối DB thành công.
* **Backup dữ liệu trước khi xóa**.

---

## 9. Lưu ý an toàn

* Luôn chạy `--dry-run` trước khi triển khai thật.
* Backup DB và recordings.
* Giới hạn quyền user DB/API để giảm rủi ro.
* Kiểm tra định kỳ log `/var/log/auto_clear_records.log`.

---

## 10. Liên hệ & hỗ trợ

Người phụ trách script: \[Phạm Minh Thảo]
Email: \[[minhthaopham230104@gmail.com](mailto:minhthaopham230104@gmail.com)]

---
