# Tự động xóa ghi âm và CDR từ danh sách Google Sheet

## Giới thiệu
Hệ thống này được thiết kế để tự động xóa **ghi âm** và **CDR** của các máy ảo (VM) dựa trên danh sách được lưu trên **Google Sheets**.  
Điều này giúp loại bỏ thao tác thủ công, giảm rủi ro sai sót và tối ưu hóa quy trình vận hành.

## Luồng hoạt động
![Luồng hoạt động](cron_clear_recordings_cdr_flow.png)

### Mô tả luồng
1. **Crontab** khởi chạy script bash định kỳ (ví dụ: 03:00 hàng ngày).
2. Script **tải danh sách tenant từ Google Sheets** (xuất sang CSV).
3. Đọc từng dòng để lấy thông tin tenant + số ngày lưu trữ.
4. Xóa ghi âm trong thư mục `/usr/local/freeswitch/recordings/<tenant>/archive/...` nếu quá hạn.
5. Gọi API xóa CDR theo tenant và thời gian lưu trữ.
6. Ghi log kết quả vào file log và gửi thông báo (nếu cần).

## Script mẫu

```bash
#!/bin/bash
# ========================
# Script: auto_clear_records.sh
# Mục tiêu: Xóa ghi âm + CDR dựa trên danh sách Google Sheet
# ========================

GOOGLE_SHEET_CSV_URL="https://docs.google.com/spreadsheets/d/<ID>/export?format=csv"
RECORDINGS_BASE="/usr/local/freeswitch/recordings"
LOG_FILE="/var/log/auto_clear_records.log"
CDR_API="https://example.com/api/delete_cdr"

# Tải danh sách CSV từ Google Sheets
curl -sL "$GOOGLE_SHEET_CSV_URL" -o /tmp/tenant_list.csv

# Đọc từng dòng trong file CSV (bỏ dòng tiêu đề)
tail -n +2 /tmp/tenant_list.csv | while IFS=',' read -r TENANT DAYS_KEEP
do
    echo "[$(date)] Xử lý tenant: $TENANT, giữ lại $DAYS_KEEP ngày" | tee -a "$LOG_FILE"

    # Xóa ghi âm cũ hơn DAYS_KEEP
    find "$RECORDINGS_BASE/$TENANT/archive" -type f -mtime +$DAYS_KEEP -exec rm -f {} \;
    
    # Gọi API xóa CDR
    curl -s -X POST "$CDR_API"          -H "Content-Type: application/json"          -d "{"tenant": "$TENANT", "days": $DAYS_KEEP}" >> "$LOG_FILE"
done

echo "[$(date)] Hoàn tất quá trình" | tee -a "$LOG_FILE"
```

## Cài đặt crontab
Ví dụ, chạy script lúc **03:00** hàng ngày:
```bash
0 3 * * * /bin/bash /home/user/scripts/auto_clear_records.sh
```

## Yêu cầu
- Máy chủ cài `curl`
- Quyền truy cập đọc/ghi thư mục recordings
- API xóa CDR hoạt động

## Tác giả
- **Người thực hiện:** Minh Thảo Phạm
- **Ngày:** 2025-08-11
