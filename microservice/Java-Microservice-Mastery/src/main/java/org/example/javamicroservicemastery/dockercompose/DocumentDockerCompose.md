# 🐳 Docker Compose - Chạy ứng dụng với PostgreSQL và Redis

## Mục tiêu
Sử dụng Docker Compose để khởi động toàn bộ môi trường (Application, PostgreSQL, Redis...) chỉ với một lệnh.

---

## Khởi động các service

```bash
docker compose up -d
```

**Ý nghĩa:**
- Đọc file `docker-compose.yml`.
- Tạo và khởi động tất cả container được khai báo.
- `-d` (detached): Chạy nền, không chiếm cửa sổ terminal.

---

## Dừng và xóa các container

```bash
docker compose down
```

**Ý nghĩa:**
- Dừng tất cả container trong Compose.
- Xóa container và network được tạo bởi Compose.
- Không xóa image.
- Volume chỉ bị xóa nếu thêm tùy chọn `-v`.

---

## Khi nào sử dụng?

| Thao tác | Lệnh |
|----------|------|
| Khởi động toàn bộ hệ thống | `docker compose up -d` |
| Dừng và dọn dẹp hệ thống | `docker compose down` |

---

## Ghi nhớ

Docker Compose giúp quản lý nhiều container (ví dụ: Spring Boot, PostgreSQL, Redis...) bằng **một file cấu hình (`docker-compose.yml`)** và **hai lệnh cơ bản**:

```bash
docker compose up -d
docker compose down
```

> 💡 Nhiều IDE (IntelliJ IDEA, VS Code...) hoặc Docker Desktop có thể chạy Docker Compose bằng giao diện, nhưng bên dưới chúng vẫn thực hiện các lệnh trên.