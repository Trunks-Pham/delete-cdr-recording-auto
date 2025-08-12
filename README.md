# Tài liệu kỹ thuật: Hệ thống tự động xóa Recordings & CDR

## Giải Pháp 1: Sử dụng theo danh sách Google Sheets
## 1. Giới thiệu

Hệ thống này được thiết kế để **tự động xóa dữ liệu ghi âm (Recordings)** và/hoặc **Call Detail Records (CDR)** của các tenant trên nền tảng Autocall / FreeSWITCH, dựa trên danh sách cấu hình được lưu trữ tại **Google Sheets** hoặc file CSV.

### Điểm nổi bật

* Hỗ trợ **nhiều loại hành động**: chỉ xóa recordings, chỉ xóa CDR, xóa cả hai, hoặc bỏ qua.
* Cho phép **cấu hình mặc định** cho `CONTEXT_TYPE` và `AUTH` trong file config, chỉ ghi đè khi cần.
* Có chế độ **Dry-run** để kiểm tra trước khi xóa.
* Cấu trúc **module hóa**, dễ mở rộng sau này.

---

## 2. Lợi ích

* **Tự động hóa** thay cho thao tác thủ công.
* **Giảm rủi ro** sai sót khi vận hành.
* **Tiết kiệm dung lượng** lưu trữ và tài nguyên hệ thống.
* **Dễ quản lý** và thay đổi cấu hình.

---

## 3. Kiến trúc hệ thống

```
┌─────────────────────┐
│  Google Sheets / CSV │
└───────────┬─────────┘
            │
      (curl tải về)
            │
┌───────────▼───────────┐
│   auto_clear.sh        │
│  ├── Đọc cấu hình      │
│  ├── Lọc dữ liệu       │
│  ├── Xóa recordings    │
│  ├── Xóa CDR (ES + DB) │
└───────────┬───────────┘
            │
     (Log kết quả)
            │
┌───────────▼───────────┐
│ /var/log/auto_clear/  │
└───────────────────────┘
```

---

## 4. Cấu trúc dữ liệu

### 4.1. Google Sheet / CSV

| STT | Tenant Name                                     | Routing UUID | From Datetime       | To Datetime         | Alias of Index | API URL                                          | CONTEXT\_TYPE | AUTH      | DB Host    | DB Port | DB Name   | DB User   | DB Pass | Action | Note                |
| --- | ----------------------------------------------- | ------------ | ------------------- | ------------------- | ---------- | ------------------------------------------------ | ------------- | --------- | ---------- | ------- | --------- | --------- | ------- | ------ | ------------------- |
| 1   | [tenant1@tenant.com](mailto:tenant1@tenant.com) | tenant1\_a7843225-8806-4e3f-b183-9df24fe6b68f        | 2024-07-01 00:00:00 | 2024-08-31 23:59:59 | index1     | [http://10.10.10.5:9200](http://10.10.10.5:9200) | *(trống)*     | *(trống)* | 10.10.10.2 | 5432    | fusionpbx | fusionpbx | pass123 | both   | Xóa dữ liệu 2 tháng |

> **Lưu ý:**
>
> * `CONTEXT_TYPE` và `AUTH` có thể để trống → sẽ dùng giá trị mặc định trong file config.
> * `Action` = `recordings` / `cdr` / `both` / `none`. Nếu trống → `both`.

---

## 5. File cấu hình (`/etc/auto_clear_records.conf`)

```bash
# URL Google Sheet xuất CSV
GOOGLE_SHEET_CSV_URL="https://docs.google.com/spreadsheets/d/<ID>/export?format=csv"

# Thư mục recordings gốc
RECORDINGS_BASE="/usr/local/freeswitch/recordings"

# File log
LOG_FILE="/var/log/auto_clear_records.log"

# Thư mục tạm
TMP_CSV="/tmp/tenant_list.csv"

# Giá trị mặc định cho ElasticSearch
DEFAULT_CONTEXT_TYPE="application/json"
DEFAULT_AUTH="Basic YWRtaW46cGFzc3dvcmQ="
```

---

## 6. Luồng hoạt động

1. Crontab kích hoạt script theo lịch định sẵn.
2. Script tải danh sách CSV từ Google Sheets.
3. Đọc từng tenant và:

   * Nếu `Action=recordings` → xóa recordings cũ.
   * Nếu `Action=cdr` → xóa CDR trên ES + DB.
   * Nếu `Action=both` → làm cả hai.
4. Ghi log kết quả.
5. Xóa file tạm.

---

## 7. Script chính (`/usr/local/bin/auto_clear_records_cdr.sh`)

> **Tối ưu:**
>
> * Tách cấu hình ra file riêng.
> * Dùng hàm cho từng loại xóa.
> * Lock file chống chạy song song.
> * Dry-run in báo cáo rõ ràng.

```bash
#!/bin/bash
set -o pipefail

# Load cấu hình
source /etc/auto_clear_records.conf

# Lock file chống chạy song song
LOCK_FILE="/tmp/auto_clear_records.lock"
exec 200>"$LOCK_FILE"
flock -n 200 || { echo "Script đang chạy, thoát."; exit 1; }

# Kiểm tra chế độ dry-run
DRY_RUN=0
if [[ "$1" == "--dry-run" ]]; then
  DRY_RUN=1
  echo "=== DRY RUN mode: Không xóa dữ liệu ==="
fi

# Hàm trim
trim() { echo "$1" | sed -e 's/^[[:space:]]*//' -e 's/[[:space:]]*$//'; }

# Hàm xóa recordings
delete_recordings() {
  local dir="$RECORDINGS_BASE/$1/archive"
  local from="$2"
  if [ ! -d "$dir" ]; then
    echo " - [Recordings] Thư mục không tồn tại: $dir" | tee -a "$LOG_FILE"
    return
  fi
  echo " - [Recordings] Xóa file trước $from tại $dir" | tee -a "$LOG_FILE"
  if [[ $DRY_RUN -eq 1 ]]; then
    find "$dir" -type f ! -newermt "$from" -print | tee -a "$LOG_FILE"
  else
    find "$dir" -type f ! -newermt "$from" -delete
  fi
}

# Hàm xóa CDR trên ElasticSearch
delete_cdr_es() {
  local api="$1"
  local index="$2"
  local routing="$3"
  local from="$4"
  local to="$5"
  local ctx_type="${6:-$DEFAULT_CONTEXT_TYPE}"
  local auth="${7:-$DEFAULT_AUTH}"

  if [[ -z "$api" || -z "$index" ]]; then
    echo " - [ES] Thiếu API_URL hoặc Index" | tee -a "$LOG_FILE"
    return
  fi

  local delete_url="$api/$index/_delete_by_query"
  local routing_filter=""
  if [[ -n "$routing" ]]; then
    routing_filter=", { \"terms\": { \"_routing\": [\"$routing\"] } }"
  fi

  local json_body=$(cat <<EOF
{
  "query": {
    "bool": {
      "filter": [
        { "range": { "start_stamp": { "from": "$from", "to": "$to" } } }
        $routing_filter
      ]
    }
  }
}
EOF
)

  echo " - [ES] Xóa dữ liệu từ $from đến $to tại $delete_url" | tee -a "$LOG_FILE"
  if [[ $DRY_RUN -eq 1 ]]; then
    echo "$json_body" | tee -a "$LOG_FILE"
  else
    if [[ -n "$auth" ]]; then
      curl -s -X POST "$delete_url" -H "Content-Type: $ctx_type" -H "Authorization: $auth" -d "$json_body" >> "$LOG_FILE"
    else
      curl -s -X POST "$delete_url" -H "Content-Type: $ctx_type" -d "$json_body" >> "$LOG_FILE"
    fi
  fi
}

# Hàm xóa CDR trên DB
delete_cdr_db() {
  local host="$1"
  local port="$2"
  local name="$3"
  local user="$4"
  local pass="$5"
  local tenant="$6"
  local from="$7"
  local to="$8"

  if [[ -z "$host" || -z "$name" || -z "$user" ]]; then
    echo " - [DB] Thiếu thông tin DB" | tee -a "$LOG_FILE"
    return
  fi

  local sql="DELETE FROM v_xml_cdr WHERE start_stamp BETWEEN '$from' AND '$to' AND context = '$tenant';"
  echo " - [DB] Xóa dữ liệu trong $name@$host" | tee -a "$LOG_FILE"
  if [[ $DRY_RUN -eq 1 ]]; then
    echo "$sql" | tee -a "$LOG_FILE"
  else
    if [[ "$port" == "5432" ]]; then
      PGPASSWORD="$pass" psql -h "$host" -p "$port" -U "$user" -d "$name" -c "$sql"
    else
      mysql -h "$host" -P "$port" -u "$user" -p"$pass" "$name" -e "$sql"
    fi
  fi
}

# Tải CSV
curl -sL "$GOOGLE_SHEET_CSV_URL" -o "$TMP_CSV" || { echo "Không tải được CSV"; exit 1; }

# Xử lý từng tenant
tail -n +2 "$TMP_CSV" | while IFS=',' read -r STT TENANT_NAME ROUTING_UUID FROM_DATETIME TO_DATETIME INDEX_UUID API_URL CONTEXT_TYPE AUTH DB_HOST DB_PORT DB_NAME DB_USER DB_PASS ACTION NOTE
do
  TENANT_NAME=$(trim "$TENANT_NAME")
  ACTION=$(echo "$(trim "$ACTION")" | tr '[:upper:]' '[:lower:]')
  [[ -z "$ACTION" ]] && ACTION="both"

  echo "=== Tenant: $TENANT_NAME | Action: $ACTION ===" | tee -a "$LOG_FILE"

  case "$ACTION" in
    recordings)
      delete_recordings "$TENANT_NAME" "$FROM_DATETIME"
      ;;
    cdr)
      delete_cdr_es "$API_URL" "$INDEX_UUID" "$ROUTING_UUID" "$FROM_DATETIME" "$TO_DATETIME" "$CONTEXT_TYPE" "$AUTH"
      delete_cdr_db "$DB_HOST" "$DB_PORT" "$DB_NAME" "$DB_USER" "$DB_PASS" "$TENANT_NAME" "$FROM_DATETIME" "$TO_DATETIME"
      ;;
    both)
      delete_recordings "$TENANT_NAME" "$FROM_DATETIME"
      delete_cdr_es "$API_URL" "$INDEX_UUID" "$ROUTING_UUID" "$FROM_DATETIME" "$TO_DATETIME" "$CONTEXT_TYPE" "$AUTH"
      delete_cdr_db "$DB_HOST" "$DB_PORT" "$DB_NAME" "$DB_USER" "$DB_PASS" "$TENANT_NAME" "$FROM_DATETIME" "$TO_DATETIME"
      ;;
    none)
      echo " - Bỏ qua tenant này" | tee -a "$LOG_FILE"
      ;;
  esac

  echo "" >> "$LOG_FILE"
done

rm -f "$TMP_CSV"
echo "Hoàn tất" | tee -a "$LOG_FILE"
```

---

## 8. Cài đặt crontab

Chạy mỗi ngày lúc 03:00 sáng:

```bash
0 3 * * * /usr/local/bin/auto_clear_records_cdr.sh
```

---

## 9. Yêu cầu hệ thống

* `curl`, `find`, `psql` (PostgreSQL), `mysql` (MySQL).
* Quyền đọc/ghi recordings.
* API ElasticSearch hoạt động.
* Kết nối DB thành công.
* **Backup trước khi xóa**.

---

## 10. Mở rộng trong tương lai

* Hỗ trợ API JSON thay CSV.
* Gửi báo cáo email sau khi chạy.
* Thêm metric Prometheus để giám sát số lượng bản ghi đã xóa.
* Tích hợp xác thực OAuth cho Google Sheets.

---

Tôi đã làm version này **tối ưu** để:

* Tách cấu hình → dễ thay đổi.
* Code có hàm riêng → dễ bảo trì/mở rộng.
* Lock file tránh chạy song song.
* Có default `CONTEXT_TYPE` và `AUTH`.

---

### **2. Kiến trúc đề xuất**

```
┌─────────────┐       ┌──────────────────────┐
│   Jenkins    │       │ Google Sheets / CSV  │
│   UI + API   │<─────>│ (Tùy chọn tải sẵn)   │
└─────┬───────┘       └──────────────────────┘
      │
      │ Trigger (Manual / Schedule)
      │
┌─────▼──────────────────────────────┐
│ Jenkins Pipeline (Groovy Script)   │
│  ├── Nhập params:                  │
│  │    - Tenant / Date range        │
│  │    - Action type (recordings/cdr)│
│  │    - CSV upload (optional)      │
│  ├── Gọi shell script               │
│  ├── Lưu log Jenkins Console        │
└─────┬──────────────────────────────┘
      │
      ▼
┌──────────────┐
│ auto_clear.sh│
│ (phiên bản   │
│ hiện tại)    │
└──────────────┘
```

---

### **3. Cách triển khai**

#### **3.1. Tạo Jenkins Job (Pipeline)**

* **Loại**: Pipeline (hoặc Freestyle job nếu đơn giản)
* **Tích hợp tham số**:

  * `ACTION_TYPE` → recordings / cdr / both
  * `TENANT_NAME` → nhập tay hoặc all
  * `FROM_DATETIME` và `TO_DATETIME`
  * **Upload file CSV** (Jenkins có plugin *File Parameter*)
  * **Dry-run** checkbox

Ví dụ code Jenkinsfile:

```groovy
pipeline {
    agent any
    parameters {
        choice(name: 'ACTION_TYPE', choices: ['recordings', 'cdr', 'both'], description: 'Loại dữ liệu cần xóa')
        string(name: 'TENANT_NAME', defaultValue: '', description: 'Tên tenant (để trống để lấy từ CSV)')
        string(name: 'FROM_DATETIME', defaultValue: '2024-07-01 00:00:00', description: 'Thời gian bắt đầu')
        string(name: 'TO_DATETIME', defaultValue: '2024-08-01 23:59:59', description: 'Thời gian kết thúc')
        booleanParam(name: 'DRY_RUN', defaultValue: true, description: 'Chạy ở chế độ dry-run')
        file(name: 'TENANT_FILE', description: 'CSV chứa danh sách tenant (tùy chọn)')
    }
    stages {
        stage('Prepare') {
            steps {
                sh '''
                echo "Chuẩn bị môi trường..."
                mkdir -p /tmp/jenkins_clear
                if [ -f "$TENANT_FILE" ]; then
                    cp "$TENANT_FILE" /tmp/tenant_list.csv
                fi
                '''
            }
        }
        stage('Execute Script') {
            steps {
                sh '''
                /usr/local/bin/auto_clear_records_cdr.sh \
                    $( [ "$DRY_RUN" = "true" ] && echo "--dry-run" ) \
                    --action "$ACTION_TYPE" \
                    --tenant "$TENANT_NAME" \
                    --from "$FROM_DATETIME" \
                    --to "$TO_DATETIME"
                '''
            }
        }
    }
}
```

---

#### **3.2. Điều chỉnh Script**

* Thêm **tùy chọn CLI** để nhận input từ Jenkins thay vì chỉ đọc CSV từ Google Sheets.
* Ưu tiên:

  * Nếu Jenkins truyền file CSV → dùng file đó.
  * Nếu Jenkins truyền tham số tenant/date → chạy trực tiếp.
  * Nếu không truyền gì → fallback về Google Sheets CSV.

Ví dụ:

```bash
--action both
--tenant tenant1
--from "2024-07-01 00:00:00"
--to "2024-08-01 23:59:59"
--dry-run
--csv /tmp/tenant_list.csv
```

---

#### **3.3. Lợi ích khi thêm Jenkins**

* **Audit log**: Biết ai chạy, khi nào, kết quả ra sao.
* **UI thân thiện**: Không cần SSH vào server.
* **Linh hoạt**: Có thể chạy ad-hoc cho 1 tenant hoặc nhóm tenant.
* **Bảo mật**: Jenkins có thể quản lý credential API/DB thay vì lưu ở file config thô.

---

#### **3.4. Bảo mật**

* Dùng Jenkins **Credentials Plugin** để lưu:

  * DB password
  * API token ElasticSearch
* Jenkins pipeline chỉ inject vào môi trường runtime → không xuất hiện trong log.

---

### **4. Lịch chạy**

* Jenkins vẫn có thể chạy **theo lịch cron** (VD: 03:00 mỗi ngày) nhưng quản lý ngay trên UI.
* Có thể kết hợp:

  * Jenkins job **tự động** mỗi ngày.
  * Jenkins job **thủ công** khi cần xóa đặc biệt.

---
