# Dockerfile tối ưu cho Spring Boot Java 21 (Multi-stage Build)

```dockerfile
# ==========================================================
# Stage 1: Build Application
# Mục đích:
# - Compile source code Java.
# - Download dependency Maven.
# - Sinh ra file .jar.
# Image ở stage này có Maven + JDK nên khá nặng.
# ==========================================================

# Sử dụng image Maven đã tích hợp JDK 21.
# Đặt tên stage là "builder" để có thể tham chiếu ở stage sau.
FROM maven:3.9.9-eclipse-temurin-21 AS builder

# Thiết lập thư mục làm việc là /app.
# Nếu chưa tồn tại Docker sẽ tự tạo.
# Các lệnh COPY và RUN phía dưới sẽ thực hiện trong thư mục này.
WORKDIR /app

# Chỉ copy pom.xml trước.
# Lý do:
# Docker cache theo từng layer.
# pom.xml ít thay đổi hơn source code nên có thể tái sử dụng cache.
COPY pom.xml .

# Download toàn bộ dependency được khai báo trong pom.xml.
# Maven sẽ tải:
# - dependency
# - plugin
# - parent pom
# - transitive dependency
#
# Các file được lưu vào:
# /root/.m2/repository
#
# Nếu pom.xml không thay đổi thì Docker sẽ sử dụng cache layer này,
# không phải download lại dependency.
RUN mvn dependency:go-offline

# Copy toàn bộ source code vào container.
# Lúc này Docker mới copy thư mục src.
#
# Nếu chỉ sửa source thì chỉ layer này bị thay đổi.
COPY src ./src

# Build project.
#
# mvn clean
#  - Xóa thư mục target cũ.
#
# package
#  - Compile source.
#  - Chạy plugin.
#  - Đóng gói thành file jar.
#
# -DskipTests
#  - Không chạy Unit Test để build nhanh hơn.
#
# Kết quả:
#
# /app/target/demo.jar
#
RUN mvn clean package -DskipTests



# ==========================================================
# Stage 2: Runtime
# Mục đích:
# Chỉ chạy ứng dụng.
# Không cần Maven.
# Không cần source code.
# Chỉ cần JRE + file jar.
# ==========================================================

# Bắt đầu một stage hoàn toàn mới.
#
# Stage này KHÔNG kế thừa filesystem của stage builder.
#
# Chỉ có:
# - Linux
# - Java Runtime (JRE)
#
# Không có:
# - Maven
# - Source code
# - .m2
# - target
#
FROM eclipse-temurin:21-jre

# Thiết lập thư mục làm việc.
# Nếu chưa có thì Docker tự tạo.
#
# Sau dòng này:
#
# /
# └── app
#
WORKDIR /app

# Copy file jar từ stage builder.
#
# --from=builder
# nghĩa là:
#
# "Lấy file từ stage tên builder."
#
# Chứ KHÔNG copy từ máy tính của bạn.
#
# Builder:
#
# /app/target/demo.jar
#
# Runtime:
#
# /app/app.jar
#
# Image cuối cùng chỉ chứa:
#
# /app/app.jar
#
# Không còn:
# - Maven
# - pom.xml
# - src
# - target
# - .m2
#
COPY --from=builder /app/target/*.jar app.jar

# Khai báo ứng dụng sẽ sử dụng cổng 8080.
#
# Đây chỉ là metadata.
#
# Docker KHÔNG tự mở cổng.
#
# Muốn truy cập từ máy host vẫn cần:
#
# docker run -p 8080:8080 image-name
#
EXPOSE 8080

# Lệnh mặc định khi container khởi động.
#
# Docker sẽ thực hiện:
#
# cd /app
# java -jar app.jar
#
# Sau đó:
#
# Spring Boot khởi động
# ↓
# Embedded Tomcat khởi động
# ↓
# Lắng nghe tại cổng 8080
#
# Nếu tiến trình Java kết thúc
# thì container cũng sẽ dừng.
#
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## 📂 Dockerfile

Dockerfile đầy đủ:
[Dockerfile](../../../../../../../Dockerfile)
---

# Luồng thực thi khi build Image

```text
docker build .

        │
        ▼

=========================
Stage 1 : Builder
=========================

FROM maven:3.9.9-eclipse-temurin-21
        │
        ▼
Tạo container build tạm
        │
        ▼
WORKDIR /app
        │
        ▼
COPY pom.xml
        │
        ▼
RUN mvn dependency:go-offline
        │
        ├── Đọc pom.xml
        ├── Download dependency
        ├── Download plugin
        ├── Download parent pom
        └── Lưu vào /root/.m2/repository
        │
        ▼
COPY src
        │
        ▼
RUN mvn clean package -DskipTests
        │
        ├── Compile source
        ├── Đóng gói
        └── Sinh file target/app.jar
        │
        ▼
Kết thúc Stage Builder



=========================
Stage 2 : Runtime
=========================

FROM eclipse-temurin:21-jre
        │
        ▼
Tạo image Runtime mới
        │
        ▼
WORKDIR /app
        │
        ▼
COPY --from=builder target/app.jar
        │
        ▼
/app/app.jar
        │
        ▼
EXPOSE 8080
        │
        ▼
ENTRYPOINT
        │
        ▼
Image hoàn chỉnh
```

---

# Luồng khi chạy Container

```text
docker run -p 8080:8080 my-app

        │
        ▼

Khởi tạo Container
        │
        ▼

WORKDIR /app
        │
        ▼

ENTRYPOINT

java -jar app.jar

        │
        ▼

JVM khởi động
        │
        ▼

Spring Boot khởi động
        │
        ▼

Khởi tạo Bean
        │
        ▼

Embedded Tomcat khởi động
        │
        ▼

Bind Port 8080
        │
        ▼

Container Running
```

---

# Cơ chế Docker Layer Cache

```text
Lần build đầu

COPY pom.xml
        │
        ▼
Download dependency (30s)
        │
        ▼
COPY src
        │
        ▼
Compile (10s)

Tổng: ~40s


=====================================


Chỉ sửa UserService.java

COPY pom.xml
        │
        ▼
(Cache)

RUN mvn dependency:go-offline
        │
        ▼
(Cache)

COPY src
        │
        ▼
(Rebuild)

RUN mvn package
        │
        ▼
(Rebuild)

Tổng: ~10s


=====================================


Nếu sửa pom.xml

COPY pom.xml
        │
        ▼
(Cache Invalid)

RUN mvn dependency:go-offline
        │
        ▼
Download dependency mới
        │
        ▼
COPY src
        │
        ▼
Compile
```

---

# Kết quả cuối cùng của Multi-stage Build

```text
Builder Image

├── Maven
├── JDK
├── pom.xml
├── src
├── target
├── /root/.m2
└── app.jar

            │
            │ COPY app.jar
            ▼

Runtime Image

├── JRE
└── /app
    └── app.jar


=> Image Runtime nhỏ hơn rất nhiều vì chỉ chứa JRE và file jar cần thiết để chạy ứng dụng.
```