# Báo Cáo Thực Hành: Xử Lý Xung Đột Phức Tạp Trong Quá Trình Rebase

## 1. Giới thiệu & Bối cảnh

Khi rebase nhánh `feature-api` lên `main`, Git phát sinh **2 lần xung đột** (conflict) tại file `config.json` — một lần cho mỗi commit trong nhánh tính năng. Báo cáo này mô tả chi tiết từng chặng phát sinh xung đột, cách phân tích và giải quyết thủ công.

---

## 2. Thiết lập môi trường thử nghiệm

### Sơ đồ phân nhánh trước khi rebase

```
* 89ba166 add env config          ← main (HEAD)
* 84473c6 update port on main
| * eed2578 feat: enable debug    ← feature-api (HEAD)
| * d7a3f3b feat: change port
|/
* 3666572 init config             ← điểm tách nhánh (base)
```

### Trạng thái `config.json` ở mỗi nhánh

| Trường | Base (init) | `main` (hiện tại) | `feature-api` (commit cuối) |
|--------|-------------|-------------------|-----------------------------|
| `port` | `8080` | `8081` | `9000` |
| `debug` | `false` | `false` | `true` |
| `env` | *(không có)* | `"production"` | *(không có)* |

---

## 3. Các bước thực hiện

### Bước 1: Tạo cấu trúc commit

```bash
# Tạo repo và commit ban đầu (trên nhánh main)
git init
echo '{ "port": 8080, "debug": false }' > config.json
git add . && git commit -m "init config"

# Tạo nhánh feature-api
git checkout -b feature-api

# Commit 1 trên feature-api: đổi port → 9000
git commit -m "feat: change port"

# Commit 2 trên feature-api: bật debug → true
git commit -m "feat: enable debug"

# Quay lại main, thêm 2 commit mới
git checkout main
git commit -m "update port on main"   # port → 8081
git commit -m "add env config"        # thêm "env": "production"
```

---

### Bước 2: Bắt đầu Rebase

```bash
git checkout feature-api
git rebase master
```

---

### Chặng 1/2 — Xung đột khi áp commit `feat: change port`

**Thông báo lỗi:**
```
Auto-merging config.json
CONFLICT (content): Merge conflict in config.json
error: could not apply d7a3f3b... feat: change port
hint: Resolve all conflicts manually, mark them as resolved with
hint: "git add/rm <conflicted_files>", then run "git rebase --continue".
```

**Nội dung file `config.json` khi có conflict markers:**

```
{
<<<<<<< HEAD
  "port": 8081,
  "debug": false,
  "env": "production"
=======
  "port": 9000,
  "debug": false
>>>>>>> d7a3f3b (feat: change port)
}
```

**Phân tích xung đột:**

| | Nội dung |
|-|---------|
| `HEAD` (main) | `port: 8081`, có thêm `env: "production"` |
| commit feature-api | `port: 9000`, chưa có `env` |
| Nguồn gốc xung đột | Cả hai nhánh cùng sửa dòng `"port"` từ `8080` ban đầu |

**Quyết định giải quyết:** Giữ `port: 9000` của feature-api (đây là thay đổi có chủ ý), đồng thời giữ `env: "production"` của main (không xung đột về ý nghĩa):

```json
{
  "port": 9000,
  "debug": false,
  "env": "production"
}
```

**Lệnh hoàn tất chặng 1:**

```bash
git add config.json
git rebase --continue
```

**Output:**
```
[detached HEAD 98f9f3c] feat: change port
 1 file changed, 1 insertion(+), 1 deletion(-)
```

---

### Chặng 2/2 — Xung đột khi áp commit `feat: enable debug`

**Thông báo lỗi:**
```
Auto-merging config.json
CONFLICT (content): Merge conflict in config.json
error: could not apply eed2578... feat: enable debug
```

**Nội dung file `config.json` khi có conflict markers:**

```
{
  "port": 9000,
<<<<<<< HEAD
  "debug": false,
  "env": "production"
=======
  "debug": true
>>>>>>> eed2578 (feat: enable debug)
}
```

**Phân tích xung đột:**

| | Nội dung |
|-|---------|
| `HEAD` (sau chặng 1) | `debug: false`, có `env: "production"` |
| commit feature-api | `debug: true`, thiếu `env` |
| Nguồn gốc xung đột | Git không biết giữ `env` hay theo feature-api |

**Quyết định giải quyết:** Giữ `debug: true` của feature-api (mục tiêu của commit này), đồng thời giữ `env: "production"` của main:

```json
{
  "port": 9000,
  "debug": true,
  "env": "production"
}
```

**Lệnh hoàn tất chặng 2:**

```bash
git add config.json
git rebase --continue
```

**Output:**
```
[detached HEAD 20602a6] feat: enable debug
 1 file changed, 1 insertion(+), 1 deletion(-)
Successfully rebased and updated refs/heads/feature-api.
```

---

## 4. Kết quả sau khi Rebase hoàn tất

### `git log --graph --oneline`

```
* 20602a6 feat: enable debug
* 98f9f3c feat: change port
* 89ba166 add env config
* 84473c6 update port on main
* 3666572 init config
```

Lịch sử **thẳng tắp hoàn toàn** — không có merge commit phụ. Các commit của `feature-api` nằm nối tiếp ngay sau commit mới nhất của `main`.

### Nội dung `config.json` cuối cùng

```json
{
  "port": 9000,
  "debug": true,
  "env": "production"
}
```

---

## 5. Tổng kết

### So sánh Rebase vs Merge khi xử lý xung đột

| Tiêu chí | Rebase | Merge |
|----------|--------|-------|
| Số lần giải quyết conflict | Mỗi commit một lần (2 lần) | Một lần duy nhất |
| Lịch sử sau khi xong | Thẳng tắp, tuyến tính | Có thêm merge commit |
| Phù hợp khi | Trước khi push lên remote | Sau khi đã push |
| Rủi ro | Phải giải quyết conflict nhiều lần | Lịch sử phức tạp hơn |

### Bài học rút ra

- Trong rebase, mỗi commit được áp lại **tuần tự** lên nhánh đích — xung đột có thể xảy ra ở **từng commit** riêng lẻ.
- Khi giải quyết conflict, cần hiểu **ý định của cả hai bên** để tích hợp đúng, không làm mất thay đổi của nhánh chính.
- Sau mỗi lần giải quyết: `git add <file>` → `git rebase --continue` — **không** dùng `git commit`.
- Nếu muốn hủy toàn bộ: `git rebase --abort` để trở về trạng thái trước khi rebase.
