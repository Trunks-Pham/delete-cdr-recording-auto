# Tự động xóa ghi âm và CDR từ danh sách Google Sheet

## Giới thiệu

Hệ thống này được thiết kế để **tự động xóa ghi âm** và **CDR** của các máy ảo (VM) dựa trên danh sách cấu hình được lưu trên **Google Sheets** hoặc file Excel.
Cơ chế này giúp:

* Loại bỏ thao tác thủ công (manual delete).
* Giảm rủi ro sai sót khi vận hành.
* Tiết kiệm thời gian và tài nguyên lưu trữ.

---

## Lưu ý về dữ liệu CDR

Trong hệ thống Autocall/FreeSWITCH:

* **CDR (Call Detail Record)** được lưu ở **hai nơi**:

  1. **Database (DB)** — thường là PostgreSQL/MySQL dùng cho FusionPBX hoặc hệ thống quản lý.
  2. **ElasticSearch (ES)** — dùng để lưu log và hỗ trợ tìm kiếm nhanh.

* Script này sẽ **xóa cả CDR trên ES** và **xóa CDR trong DB** dựa trên khoảng thời gian (`FROM_DATETIME` - `TO_DATETIME`) và `_routing`/tenant.

* Xóa CDR trong DB là thao tác **không thể khôi phục** → cần backup trước khi chạy.

---

## Luồng hoạt động

![Luồng hoạt động](cron_clear_recordings_cdr_flow.png)

### Mô tả luồng

1. **Crontab** khởi chạy script shell định kỳ (ví dụ: 03:00 hàng ngày).
2. Script **tải danh sách tenant từ Google Sheets** (xuất sang CSV).
3. Đọc từng dòng để lấy thông tin tenant, UUID routing, thời gian xóa, thông tin DB và ES.
4. Xóa ghi âm cũ trong thư mục `/usr/local/freeswitch/recordings/<tenant>/archive/...`.
5. Gọi API ElasticSearch `_delete_by_query` để xóa dữ liệu CDR trên ES.
6. Gọi SQL để xóa CDR trong Database.
7. Ghi log kết quả và gửi thông báo nếu cần.

---

## Cấu trúc file Google Sheet / Excel

Tên file ví dụ: **delete\_cdr\_schedule.xlsx**

| STT | Tenant Name | Routing UUID                                  | From Datetime       | To Datetime         | Index UUID       | API URL                                          | DB Host    | DB Port | DB Name   | DB User   | DB Pass | Note                |
| --- | ----------- | --------------------------------------------- | ------------------- | ------------------- | ---------------- | ------------------------------------------------ | ---------- | ------- | --------- | --------- | ------- | ------------------- |
| 1   | tenant1     | tenant1\_a7843225-8806-4e3f-b183-9df24fe6b68f | 2024-07-01 00:00:00 | 2024-08-31 23:00:00 | cdr\_index\_2024 | [http://10.10.10.5:9200](http://10.10.10.5:9200) | 10.10.10.2 | 5432    | fusionpbx | fusionpbx | pass123 | Xóa dữ liệu 2 tháng |
| 2   | tenant2     | tenant2\_b1523421-8806-4e3f-b183-9df24fe6b68f | 2024-06-01 00:00:00 | 2024-06-30 23:00:00 | cdr\_index\_2024 | [http://10.10.10.5:9200](http://10.10.10.5:9200) | 10.10.10.3 | 3306    | autocall  | autocall  | pass456 | Xóa tháng 6         |

### Giải thích:

* **Routing UUID** → giá trị `_routing` trong ES.
* **From Datetime** và **To Datetime** → khoảng thời gian xóa.
* **Index UUID** → index ElasticSearch cần xóa.
* **API URL** → `http://ip:port` của ElasticSearch.
* **DB Host/Port/Name/User/Pass** → thông tin kết nối Database để xóa CDR.
* **Note** → ghi chú cho quản trị viên.

---

## Script mẫu

```bash
#!/bin/bash
# ========================
# Script: auto_clear_records_cdr.sh
# ========================

GOOGLE_SHEET_CSV_URL="https://docs.google.com/spreadsheets/d/<ID>/export?format=csv"
RECORDINGS_BASE="/usr/local/freeswitch/recordings"
LOG_FILE="/var/log/auto_clear_records.log"

# ==== Cấu hình cố định cho API ES ====
CONTEXT_TYPE="application/json"
AUTH="Basic YWRtaW46cGFzc3dvcmQ="  # hoặc Bearer token

# Tải danh sách CSV từ Google Sheets
curl -sL "$GOOGLE_SHEET_CSV_URL" -o /tmp/tenant_list.csv

# Bỏ dòng tiêu đề và đọc từng dòng
tail -n +2 /tmp/tenant_list.csv | while IFS=',' read -r TENANT_NAME ROUTING_UUID FROM_DATETIME TO_DATETIME INDEX_UUID API_URL DB_HOST DB_PORT DB_NAME DB_USER DB_PASS NOTE
do
    echo "[$(date)] Xử lý tenant: $TENANT_NAME ($ROUTING_UUID)" | tee -a "$LOG_FILE"

    # Xóa ghi âm
    if [ -d "$RECORDINGS_BASE/$TENANT_NAME/archive" ]; then
        echo " - Xóa ghi âm cũ hơn $FROM_DATETIME" | tee -a "$LOG_FILE"
        find "$RECORDINGS_BASE/$TENANT_NAME/archive" -type f ! -newermt "$FROM_DATETIME" -exec rm -f {} \;
    else
        echo " - Không tìm thấy thư mục recordings cho $TENANT_NAME" | tee -a "$LOG_FILE"
    fi

    # Xóa CDR trên ES
    DELETE_URL="$API_URL/$INDEX_UUID/_delete_by_query"
    JSON_BODY=$(cat <<EOF
{
  "track_total_hits": true,
  "query": {
    "bool": {
      "filter": [
        {
          "range": {
            "start_stamp": {
              "from": "$FROM_DATETIME",
              "to": "$TO_DATETIME",
              "include_lower": true,
              "include_upper": true
            }
          }
        },
        {
          "terms": {
            "_routing": ["$ROUTING_UUID"]
          }
        }
      ]
    }
  }
}
EOF
)
    echo " - Gọi API xóa CDR ES" | tee -a "$LOG_FILE"
    curl -s -X POST "$DELETE_URL" \
         -H "Content-Type: $CONTEXT_TYPE" \
         -H "Authorization: $AUTH" \
         -d "$JSON_BODY" >> "$LOG_FILE"

    # Xóa CDR trong DB
    echo " - Xóa CDR trong DB: $DB_NAME@$DB_HOST" | tee -a "$LOG_FILE"
    if [[ "$DB_PORT" == "5432" ]]; then
        PGPASSWORD="$DB_PASS" psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -c \
        "DELETE FROM v_xml_cdr WHERE start_stamp BETWEEN '$FROM_DATETIME' AND '$TO_DATETIME' AND context = '$TENANT_NAME';" >> "$LOG_FILE"
    else
        mysql -h "$DB_HOST" -P "$DB_PORT" -u "$DB_USER" -p"$DB_PASS" "$DB_NAME" -e \
        "DELETE FROM v_xml_cdr WHERE start_stamp BETWEEN '$FROM_DATETIME' AND '$TO_DATETIME' AND context = '$TENANT_NAME';" >> "$LOG_FILE"
    fi

    echo "" >> "$LOG_FILE"
done

echo "[$(date)] Hoàn tất quá trình" | tee -a "$LOG_FILE"
```

---

## Cài đặt crontab

```bash
0 3 * * * /bin/bash /home/user/scripts/auto_clear_records_cdr.sh
```

---

## Yêu cầu

* Máy chủ cài `curl`, `find`, `psql` (nếu dùng PostgreSQL), `mysql` (nếu dùng MySQL)
* Quyền truy cập đọc/ghi thư mục recordings
* API ElasticSearch `_delete_by_query` hoạt động
* Kết nối DB thành công từ server chạy script
* Backup dữ liệu trước khi xóa

---
