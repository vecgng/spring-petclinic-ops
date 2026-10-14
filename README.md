# Spring PetClinic Operations Lab

## 项目简介

本项目基于 Spring 官方开源项目 Spring PetClinic，完成一套 Java Web 应用的容器化部署与 Linux 运维实践。

原始业务项目：https://github.com/spring-projects/spring-petclinic

本仓库主要保存个人编写和整理的部署配置及运维文档，不包含 Spring PetClinic 的业务源代码。

## 技术环境

- Rocky Linux 9.8
- Docker Engine / Docker Compose
- Spring Boot / Java 17 / Maven
- MySQL 9.7
- Nginx
- SELinux / firewalld
- Git / GitHub

## 部署架构

Windows 浏览器 → Nginx（8082）→ Spring Boot（8081 → 容器内 8080）→ MySQL（容器内 3306）

其中，Spring Boot 和 MySQL 通过 Docker Compose 管理，Nginx 由 systemd 管理。

## 已实现功能

- 使用 Maven 构建 Spring Boot JAR 文件
- 使用 Dockerfile 构建应用镜像
- 通过 Docker Compose 管理应用和数据库
- 配置 Nginx 反向代理
- 配置 SELinux HTTP 端口及网络连接权限
- 使用 Docker Volume 持久化 MySQL 数据
- 使用 mysqldump 备份和恢复数据库
- 为 Spring Boot 和 MySQL 配置健康检查
- 验证应用读写数据库及容器重建后的数据完整性

## 部署步骤

### 1. 获取业务源码

本仓库不包含 Spring PetClinic 的业务源代码，需要先从官方 GitHub 仓库获取。

```bash
git clone https://github.com/spring-projects/spring-petclinic.git
cd spring-petclinic
```

### 2. 使用 Maven 构建 JAR

可以使用包含 Maven 和 Java 17 的 Docker 镜像进行构建，无须在宿主机安装 Java 和 Maven。

```bash
docker run --rm \
  -v "$PWD":/app \
  -v "$HOME/.m2":/root/.m2 \
  -w /app \
  maven:3.9-eclipse-temurin-17 \
  mvn package -DskipTests
```

构建成功后，应生成：

`target/spring-petclinic-4.0.0-SNAPSHOT.jar`

### 3. 构建应用镜像

进入本运维仓库所在目录，将构建好的 JAR 复制到该目录的 `target/` 下。

```bash
mkdir -p target

cp ../spring-petclinic/target/spring-petclinic-4.0.0-SNAPSHOT.jar target/

docker build -t spring-petclinic:1.0 .
```

上述命令假定两个仓库位于同一父目录下。

### 4. 配置数据库环境变量

```bash
cp .env.example .env
```

编辑 `.env`，为 `MYSQL_ROOT_PASSWORD` 和 `MYSQL_PASSWORD` 分别设置强密码。不要将 `.env` 上传到 GitHub。

### 5. 启动应用与数据库

```bash
docker compose up -d app
docker compose ps
```

Compose 将创建 MySQL 数据卷，并在 MySQL 健康检查通过后启动 Spring Boot。

### 6. 检查服务状态

```bash
docker compose logs --tail 30 app
docker compose logs --tail 30 mysql
docker compose ps
```

Spring Boot 应用使用宿主机的 `127.0.0.1:8081`，MySQL 使用宿主机的 `127.0.0.1:3307`。

Nginx 反向代理、安全策略和外部访问入口需要根据服务器环境另外配置。


