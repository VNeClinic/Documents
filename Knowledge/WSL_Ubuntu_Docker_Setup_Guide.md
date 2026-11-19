# Hướng dẫn cài đặt WSL, Ubuntu và Docker CLI trên Windows

## 1. Cài đặt WSL và Ubuntu

### Bước 1: Mở Windows PowerShell với quyền Administrator

- Nhấn Start Menu.
- Tìm **Windows PowerShell**.
- Nhấp chuột phải và chọn **Run as administrator**.

### Bước 2: Cập nhật WSL và cài đặt Ubuntu

Chạy các lệnh sau trong PowerShell:

```powershell
wsl --update
wsl --install -d Ubuntu
```

---

## 2. Thiết lập phiên bản WSL mặc định là WSL2

Sau khi hoàn tất cài đặt Ubuntu, tiếp tục chạy lệnh sau trong **Windows PowerShell (Administrator)**:

> Các bản phân phối Linux được cài đặt từ thời điểm này sẽ sử dụng WSL2 làm mặc định.

```powershell
wsl --set-default-version 2
```

---

## Chuyển sang Terminal Ubuntu (WSL)

Các lệnh trong những phần tiếp theo sẽ được thực hiện trên **Ubuntu Terminal (WSL)**.

### Khởi động Ubuntu Terminal

1. Mở **Start Menu**.
2. Tìm ứng dụng **Ubuntu**.
3. Nhấp chuột phải vào **Ubuntu** và chọn **Run as administrator**.
4. Xác nhận cửa sổ Ubuntu Terminal đã khởi động thành công.

---

## 3. Cài đặt Docker CLI và Docker Engine

### Bước 1: Cập nhật repository hệ thống

```bash
sudo apt-get update
```

### Bước 2: Cài đặt các package cần thiết

```bash
sudo apt-get -y install ca-certificates curl gnupg lsb-release
```

### Bước 3: Tạo thư mục lưu Docker GPG Key

```bash
sudo mkdir -p /etc/apt/keyrings
```

### Bước 4: Tải và cài đặt Docker GPG Key

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

### Bước 5: Thêm Docker Repository

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Bước 6: Cập nhật danh sách package

```bash
sudo apt-get update
```

### Bước 7: Cài đặt Docker Engine và Docker CLI

```bash
sudo apt-get -y install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

---

## Kiểm tra cài đặt Docker

Sau khi cài đặt hoàn tất, có thể kiểm tra phiên bản Docker:

```bash
docker --version
```

Kiểm tra Docker Compose:

```bash
docker compose version
```

---

## Tóm tắt quy trình

1. Cài đặt WSL.
2. Cài đặt Ubuntu.
3. Thiết lập WSL2 làm mặc định.
4. Khởi động Ubuntu Terminal.
5. Cài đặt Docker Repository.
6. Cài đặt Docker Engine và Docker CLI.
7. Xác nhận Docker hoạt động bình thường.
