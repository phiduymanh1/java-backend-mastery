# ==========================
# Stage 1: Build Application
# ==========================
FROM maven:3.9.9-eclipse-temurin-21 AS builder
# Tạo mới nếu chưa tồn tại và nhảy vào thư mục /app
WORKDIR /app

# Copy dependency trước để tận dụng Docker cache
COPY pom.xml .
RUN mvn dependency:go-offline

# Copy source
COPY src ./src

# Build project
RUN mvn clean package -DskipTests


# ==========================
# Stage 2: Runtime
# ==========================
FROM eclipse-temurin:21-jre

WORKDIR /app

# Copy file jar từ stage build
COPY --from=builder /app/target/*.jar app.jar

# Expose port
EXPOSE 8080

# Chạy ứng dụng
ENTRYPOINT ["java", "-jar", "app.jar"]