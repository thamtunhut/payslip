# Payslip Generator

Công cụ tạo phiếu lương PDF từ file Excel bảng lương — chạy hoàn toàn trên trình duyệt (client-side), không cần server, không gửi dữ liệu lên bất kỳ đâu.

## Cách dùng

1. Mở [`index.html`](index.html) bằng trình duyệt (double-click hoặc qua GitHub Pages).
2. Tải lên file Excel bảng lương (`.xlsx`).
3. Chọn Tháng/Năm cho kỳ lương, điều chỉnh Bộ Phận/Phép Năm/Đi Lại nếu cần.
4. Xem trước từng phiếu lương (có zoom in/out).
5. Tải PDF cho từng nhân viên riêng lẻ, hoặc bấm "Tải tất cả PDF" để tải hết cùng lúc.

## Đặc điểm

- Không cần cài đặt, không cần server — mở file `index.html` là chạy được (kể cả offline).
- Tất cả thư viện xử lý (đọc Excel, xuất PDF) đã được nhúng sẵn trong file, không phụ thuộc CDN.
- Dữ liệu Excel chỉ được xử lý trong trình duyệt của bạn, không upload lên đâu cả.

## Lưu ý

Không commit file Excel bảng lương thật hoặc PDF phiếu lương đã xuất vào repo này (đã cấu hình trong `.gitignore`) vì chứa dữ liệu cá nhân (CCCD, mã số thuế, lương).
