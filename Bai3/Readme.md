# Báo Cáo Thực Hành: Xử Lý Xung Đột Phức Tạp Trong Quá Trình Rebase

## 1. Giới thiệu & Bối cảnh
Trong dự án thực tế, khi nhánh `main` liên tục cập nhật các commit mới từ thành viên khác, việc rebase nhánh tính năng (`feature-api`) lên `main` sẽ giúp lịch sử Git thẳng hàng, sạch sẽ. Tuy nhiên, nếu cả hai nhánh cùng sửa đổi những dòng giống nhau trên file cấu hình `config.json`, xung đột (conflict) sẽ phát sinh ở từng chặng commit. Báo cáo này ghi lại chi tiết quá trình giải quyết xung đột từng bước (step-by-step conflict resolution) trong tiến trình rebase.

---

## 2. Các bước tái lập bối cảnh

### Bước 1: Khởi tạo trên nhánh `main`
1. Tạo file `config.json` với nội dung ban đầu:
   ```json
   {
     "port": 8080,
     "debug": false
   }