# Spring PetClinic 故障排查记录

## 1. MySQL 容器启动失败：端口冲突

**故障现象**

Docker Compose 启动 MySQL 时出现：

```text
failed to bind host port 0.0.0.0:3306/tcp: address already in use
```

**排查过程**

使用 `ss -lntp` 检查端口，发现宿主机已有 `mysqld` 占用 3306。

**解决方法**

将 Docker MySQL 的端口映射调整为：

```yaml
ports:
  - "127.0.0.1:3307:3306"
```

**总结：** 容器内部端口可以相同，但同一宿主机地址上的端口不能被重复绑定。

## 2. Nginx 无法监听 8082：SELinux 权限限制

**故障现象**

Nginx 配置检查成功，但访问 8082 时出现 Connection refused。

错误日志：

```text
bind() to 0.0.0.0:8082 failed (13: Permission denied)
```

**排查及解决**

```bash
getenforce
sudo semanage port -l | grep '^http_port_t'
sudo semanage port -a -t http_port_t -p tcp 8082
sudo systemctl reload nginx
```

**总结：** 配置语法正确，不代表服务拥有监听端口的权限。

## 3. Nginx 返回 502 Bad Gateway

**故障现象**

Nginx 已监听 8082，但反向代理到 Spring Boot 失败。

错误日志：

```text
connect() to 127.0.0.1:8081 failed (13: Permission denied)
```

**解决方法**

```bash
sudo setsebool -P httpd_can_network_connect on
```

再次测试：

```bash
curl -I http://127.0.0.1:8082
```

**总结：** Nginx 监听端口的权限和主动连接后端的权限是两回事。该布尔开关会扩大相关 Web 服务的网络连接权限，应评估适用范围。

## 4. 添加主人成功，但页面跳转到 404

**故障现象**

通过 `8082` 访问 Spring PetClinic，提交表单后跳转到了不带端口号的地址。

**原因分析**

反向代理传递给应用的原始主机及端口信息不完整，导致重定向地址错误。

**解决方法**

完善 Nginx 请求头：

```nginx
proxy_set_header Host $http_host;
proxy_set_header X-Forwarded-Host $http_host;
proxy_set_header X-Forwarded-Port $server_port;
proxy_set_header X-Forwarded-Proto $scheme;
```

**总结：** 页面显示 404 不代表数据库写入失败。曾因重复提交产生四条主人记录，需要通过数据库查询确认实际执行结果。

## 5. MySQL 中文显示为问号

**故障现象**

浏览器显示中文正常，但 MySQL 命令行查询显示 `????`。

**排查过程**

通过 `HEX(address)` 验证数据库实际保存的是 UTF-8 中文字节。

检查发现 MySQL 命令行连接字符集为 `latin1`。

**解决方法**

```sql
SET NAMES utf8mb4;
```

**总结：** 数据库存储字符集与客户端连接字符集不同，显示乱码不一定代表数据损坏。

## 6. mysqldump 备份权限不足

**故障现象**

普通数据库用户执行备份时，遇到 `RELOAD` / `FLUSH_TABLES` 权限不足及 masking policy 读取权限错误。

**解决方法**

针对当前单机学习环境，使用：

```bash
mysqldump -u petclinic -p \
  --single-transaction \
  --set-gtid-purged=OFF \
  --no-tablespaces \
  --skip-masking-policies \
  --default-character-set=utf8mb4 \
  petclinic > petclinic.sql
```

**验证结果**

备份文件包含七张业务表和数据，成功恢复到独立测试数据库，并验证 `owners` 表恢复出 14 条记录。

**总结：** 备份需要检查执行结果，并通过独立恢复测试验证，不能只看文件是否生成。
