# Windows 下 MySQL 生产环境运维最佳实践

## 目录

1. [概述](#概述)
2. [安装与配置](#安装与配置)
3. [性能优化](#性能优化)
4. [备份与恢复](#备份与恢复)
5. [监控与告警](#监控与告警)
6. [安全加固](#安全加固)
7. [高可用方案](#高可用方案)
8. [日常维护](#日常维护)
9. [故障排查](#故障排查)
10. [总结](#总结)

---

## 概述

在生产环境中运行 MySQL 数据库是一项关键任务，需要综合考虑性能、安全性、可靠性和可维护性。本文档专门针对 **Windows Server** 平台上的 MySQL 生产环境运维，提供全面的最佳实践指南。

> **注意**：虽然 Linux 是 MySQL 最常见的部署平台，但在某些企业环境中，Windows Server 仍然是首选或唯一选择。了解如何在 Windows 上正确运维 MySQL 同样重要。

### 适用场景

- 企业级 Windows Server 上的 MySQL 部署
- 已有 Windows 环境的升级与维护
- 混合架构中的数据库节点
- 对 Windows 平台有特定要求的客户环境

---

## 安装与配置

### 1. 选择合适的版本

| 版本类型 | 适用场景 | 建议 |
|---------|---------|------|
| MySQL Community Server | 测试、开发环境 | ✅ 适合非生产环境 |
| MySQL Enterprise Server | 生产环境（付费） | ✅ 推荐生产使用 |
| Percona Server | 生产环境（免费） | ✅ 优秀的开源替代 |
| MariaDB | 生产环境（免费） | ⚠️ 需评估兼容性 |

**最佳实践**：
- 生产环境建议使用 **MySQL 8.0 LTS** 或更高版本
- 避免在生产环境使用 GA 日期不足 6 个月的新版本
- 确保版本与操作系统完全兼容

### 2. Windows 服务配置

```cmd
:: 以管理员身份运行

:: 安装 MySQL 为 Windows 服务
mysqld --install MySQL80 --defaults-file="C:\ProgramData\MySQL\MySQL Server 8.0\my.ini"

:: 设置服务启动类型
sc config MySQL80 start= auto

:: 启动服务
net start MySQL80
```

**配置要点**：
- 使用 `--defaults-file` 指定配置文件路径
- 服务账户应使用专用低权限账户，而非 Administrator
- 配置自动启动，但设置合理的延迟（可选）

### 3. my.ini 核心配置

```ini
[mysqld]
# ==================== 基本设置 ====================
port = 3306
basedir = C:/Program Files/MySQL/MySQL Server 8.0
datadir = D:/MySQL/Data
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

# ==================== 内存设置 ====================
innodb_buffer_pool_size = 4G
innodb_log_file_size = 1G
innodb_log_buffer_size = 64M
sort_buffer_size = 2M
join_buffer_size = 4M
tmp_table_size = 64M
max_heap_table_size = 64M

# ==================== 连接设置 ====================
max_connections = 500
max_connect_errors = 10000
wait_timeout = 28800
interactive_timeout = 28800

# ==================== 日志设置 ====================
log_error = D:/MySQL/Logs/mysql-error.log
general_log = OFF
general_log_file = D:/MySQL/Logs/mysql-general.log
slow_query_log = ON
slow_query_log_file = D:/MySQL/Logs/mysql-slow.log
long_query_time = 2
log_queries_not_using_indexes = ON

# ==================== InnoDB 设置 ====================
innodb_flush_method = NORMAL
innodb_flush_log_at_trx_commit = 1
innodb_file_per_table = ON
innodb_open_files = 65535
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000

# ==================== 二进制日志 ====================
server-id = 1
log_bin = D:/MySQL/BinLog/mysql-bin
binlog_format = ROW
binlog_expire_logs_seconds = 604800
max_binlog_size = 100M
sync_binlog = 1

# ==================== 其他设置 ====================
default_authentication_plugin = mysql_native_password
secure_file_priv = C:/MySQL/Exports
lower_case_table_names = 1
```

**关键配置说明**：

| 参数 | 说明 | 建议值 |
|-----|------|-------|
| `innodb_buffer_pool_size` | InnoDB 缓冲池大小 | 物理内存的 60-70% |
| `innodb_log_file_size` | Redo Log 文件大小 | 根据写入负载调整 |
| `innodb_flush_log_at_trx_commit` | 持久性级别 | 1=最高安全，2=高性能 |
| `max_connections` | 最大连接数 | 根据应用需求调整 |
| `slow_query_log` | 慢查询日志 | 生产环境必须开启 |

### 4. Windows 磁盘优化

```powershell
# 禁用磁盘索引（数据库服务器）
Set-WindowsSearchService -EnableIndex $false

# 关闭页面文件交换优化（如有足够内存）
# 在系统属性 -> 高级 -> 性能 -> 高级 -> 虚拟内存中配置

# 启用 NTFS 快速删除
fsutil behavior set DisableLastAccess 1

# 优化磁盘分区对齐（SSD/NVMe）
diskpart
select disk 0
clean
create partition primary align=1024
format fs=ntfs quick
assign
exit
```

**磁盘建议**：
- 数据文件、日志文件、备份文件使用不同物理磁盘
- SSD/NVMe 显著提升随机读写性能
- RAID 10 是数据库的理想选择
- 定期运行 `defrag` 或 `optimize-drives`

---

## 性能优化

### 1. 缓冲池优化

```sql
-- 检查缓冲池使用情况
SELECT 
    ROUND(Innodb_buffer_pool_pages_data * Innodb_page_size / 1024 / 1024, 2) AS 'Data Size (MB)',
    ROUND((Innodb_buffer_pool_pages_total - Innodb_buffer_pool_pages_free) * Innodb_page_size / 1024 / 1024, 2) AS 'Used Size (MB)',
    ROUND(Innodb_buffer_pool_pages_free * Innodb_page_size / 1024 / 1024, 2) AS 'Free Size (MB)',
    ROUND((Innodb_buffer_pool_pages_total - Innodb_buffer_pool_pages_free) * 100.0 / Innodb_buffer_pool_pages_total, 2) AS 'Hit Rate (%)'
FROM information_schema.INNODB_METRICS
WHERE name IN ('buffer_pool_pages_total', 'buffer_pool_pages_free', 'buffer_pool_pages_data');

-- 更简单的命中率查询
SHOW STATUS LIKE 'Innodb_buffer_pool_read%';
```

**计算命中率**：
```
命中率 = (1 - reads_from_disk / requests) * 100%
目标：> 99%
```

### 2. 查询优化

```sql
-- 查看最耗时的查询
SELECT 
    DIGEST_TEXT AS query,
    COUNT_STAR AS exec_count,
    ROUND(AVG_TIMER_WAIT / 1000000000, 2) AS avg_time_ms,
    ROUND(SUM_TIMER_WAIT / 1000000000, 2) AS total_time_s,
    ROUND(MAX_TIMER_WAIT / 1000000000, 2) AS max_time_ms
FROM performance_schema.events_statements_summary_by_digest
ORDER BY total_time_s DESC
LIMIT 20;

-- 查看未使用索引的查询
SELECT 
    SCHEMA_NAME,
    DIGEST_TEXT,
    COUNT_STAR
FROM performance_schema.events_statements_summary_by_digest
WHERE INDEXES_IS_USED = 'NO'
ORDER BY COUNT_STAR DESC;
```

### 3. 索引优化

```sql
-- 查找缺少索引的表
SELECT 
    TABLE_SCHEMA,
    TABLE_NAME,
    TABLE_ROWS,
    DATA_LENGTH,
    INDEX_LENGTH
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'performance_schema', 'mysql')
ORDER BY DATA_LENGTH DESC;

-- 分析表的使用情况
ANALYZE TABLE your_database.your_table;

-- 检查索引使用情况
SELECT 
    OBJECT_SCHEMA,
    OBJECT_NAME,
    INDEX_NAME,
    COUNT_STAR,
    COUNT_READ,
    COUNT_WRITE
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE INDEX_NAME IS NOT NULL
ORDER BY COUNT_READ + COUNT_WRITE DESC;
```

**索引最佳实践**：
- 为频繁查询的列创建索引
- 避免在低基数列上创建索引
- 定期执行 `ANALYZE TABLE` 更新统计信息
- 使用 `EXPLAIN` 分析查询计划
- 复合索引遵循最左前缀原则

### 4. Windows 特定优化

```powershell
# 设置电源模式为高性能
powercfg -setactive 8c5e7f1e-26ae-412e-8f0c-27a93e1b5d17

# 禁用 IPv6（如果不需要）
Disable-NetAdapterBinding -Name "*" -ComponentID ms_tcpip6

# 调整 TCP 参数
netsh int tcp set global autotuninglevel=normal
netsh int tcp set global rss=enabled

# 优化网络 MTU
netsh interface ipv4 show subinterfaces
```

---

## 备份与恢复

### 1. 备份策略

| 备份类型 | 频率 | 保留时间 | 用途 |
|---------|------|---------|------|
| 全量备份 | 每周 | 4 周 | 完整恢复 |
| 增量备份 | 每天 | 7 天 | 点恢复 |
| 二进制日志 | 实时 | 7 天 | 时间点恢复 |

### 2. mysqldump 备份脚本

```batch
@echo off
REM backup_full.bat - 全量备份脚本

set BACKUP_DIR=D:\MySQL\Backups
set DATE=%date:~0,4%%date:~5,2%%date:~8,2%
set TIME=%time:~0,2%%time:~3,2%%time:~6,2%
set TIME=%TIME: =0%
set TIMESTAMP=%DATE%_%TIME%

echo Starting full backup at %TIMESTAMP%...

"C:\Program Files\MySQL\MySQL Server 8.0\bin\mysqldump.exe" ^
    --single-transaction ^
    --routines ^
    --triggers ^
    --events ^
    --master-data=2 ^
    --flush-logs ^
    --hex-blob ^
    --all-databases ^
    --user=root ^
    --password=YourPassword ^
    > "%BACKUP_DIR%\full_backup_%TIMESTAMP%.sql"

if %ERRORLEVEL% EQU 0 (
    echo Backup completed successfully.
    :: 压缩备份文件
    powershell -Command "Compress-Archive -Path '%BACKUP_DIR%\full_backup_%TIMESTAMP%.sql' -DestinationPath '%BACKUP_DIR%\full_backup_%TIMESTAMP%.zip'"
    del "%BACKUP_DIR%\full_backup_%TIMESTAMP%.sql"
    
    :: 删除 30 天前的备份
    powershell -Command "Get-ChildItem '%BACKUP_DIR%\*.zip' | Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-30) } | Remove-Item"
) else (
    echo Backup failed!
)

echo Backup finished at %date% %time%
```

### 3. XtraBackup 热备份（推荐）

```batch
@echo off
REM xtrabackup_incremental.bat - 增量备份脚本

set BACKUP_DIR=D:\MySQL\Backups
set DATE=%date:~0,4%%date:~5,2%%date:~8,2%
set TIMESTAMP=%DATE%_%time:~0,2%%time:~3,2%%time:~6,2%
set TIMESTAMP=%TIMESTAMP: =0%

echo Starting incremental backup at %TIMESTAMP%...

"C:\Program Files\Percona\XtraBackup\bin\xtrabackup.exe" ^
    --user=root ^
    --password=YourPassword ^
    --backup ^
    --target-dir="%BACKUP_DIR%\incremental_%TIMESTAMP%" ^
    --slave-info ^
    --parallel=4 ^
    --compress ^
    --compress-threads=4

if %ERRORLEVEL% EQU 0 (
    echo Incremental backup completed successfully.
) else (
    echo Incremental backup failed!
)
```

### 4. 恢复流程

```sql
-- 步骤 1：停止 MySQL 服务
net stop MySQL80

-- 步骤 2：准备备份（还原前必须执行）
"C:\Program Files\Percona\XtraBackup\bin\xtrabackup.exe" ^
    --prepare ^
    --target-dir="D:\MySQL\Backups\incremental_20261001_120000"

-- 步骤 3：复制数据文件
xcopy /E /Y "D:\MySQL\Backups\incremental_20261001_120000\*" "D:\MySQL\Data\"

-- 步骤 4：启动服务
net start MySQL80

-- 步骤 5：验证数据
SELECT COUNT(*) FROM your_database.your_table;
```

### 5. 自动化备份计划

```powershell
# 创建 Windows 任务计划程序任务
$action = New-ScheduledTaskAction -Execute "cmd.exe" -Argument "/c D:\MySQL\Scripts\backup_full.bat"
$trigger = New-ScheduledTaskTrigger -Daily -At 2:00AM
$settings = New-ScheduledTaskSettingsSet -StartOnlyIfIdle -WakeToRun
Register-ScheduledTask -TaskName "MySQL Full Backup" -Action $action -Trigger $trigger -Settings $settings -Description "Weekly full backup of MySQL databases"
```

---

## 监控与告警

### 1. 关键监控指标

| 指标 | 正常范围 | 警告阈值 | 严重阈值 |
|-----|---------|---------|---------|
| CPU 使用率 | < 70% | 70-85% | > 85% |
| 内存使用率 | < 80% | 80-90% | > 90% |
| 磁盘使用率 | < 70% | 70-85% | > 85% |
| 连接数 | < 80% max | 80-90% | > 90% |
| QPS | 基线 ±20% | ±30% | ±50% |
| TPS | 基线 ±20% | ±30% | ±50% |
| 慢查询数 | 0-5/min | 5-20/min | > 20/min |
| 主从延迟 | 0-1s | 1-5s | > 5s |

### 2. 使用 PowerShell 监控脚本

```powershell
# monitor_mysql.ps1 - MySQL 监控脚本

$server = "localhost"
$user = "root"
$password = "YourPassword"
$db = "information_schema"

$connStr = "Server=$server;Uid=$user;Pwd=$password;"

try {
    # 获取连接状态
    $conn = New-Object System.Data.SqlClient.SqlConnection($connStr)
    $conn.Open()
    
    # 检查 MySQL 进程
    $process = Get-Process -Name "mysqld" -ErrorAction SilentlyContinue
    
    if ($process) {
        Write-Host "MySQL Process: Running" -ForegroundColor Green
        Write-Host "CPU Usage: $($process.CPU)% " -NoNewline
        if ($process.CPU -gt 80) {
            Write-Host "(HIGH)" -ForegroundColor Red
        } else {
            Write-Host ""
        }
        
        # 检查内存使用
        $memUsage = $process.WorkingSet64 / 1GB
        Write-Host "Memory Usage: $([math]::Round($memUsage, 2)) GB"
    } else {
        Write-Host "MySQL Process: NOT RUNNING" -ForegroundColor Red
        # 发送告警
        Send-MailMessage -To "admin@example.com" -Subject "MySQL Alert" -Body "MySQL is not running!" -SmtpServer "smtp.example.com"
    }
    
    $conn.Close()
} catch {
    Write-Host "Error: $_" -ForegroundColor Red
}
```

### 3. 使用 Prometheus + Grafana 监控

```yaml
# prometheus.yml 配置示例
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'mysql'
    static_configs:
      - targets: ['localhost:9104']
    metrics_path: '/metrics'
```

**推荐的 Exporter**：
- **mysqld_exporter**：MySQL 官方 exporter
- **node_exporter**：系统指标
- **blackbox_exporter**：可用性监控

### 4. 自定义告警规则

```yaml
# alert_rules.yml
groups:
  - name: mysql_alerts
    rules:
      - alert: MySQLDown
        expr: up{job="mysql"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MySQL instance is down"
          
      - alert: HighConnections
        expr: mysql_global_status_threads_connected / mysql_global_variables_max_connections * 100 > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MySQL connection usage is above 90%"
          
      - alert: SlowQueries
        expr: rate(mysql_global_status_slow_queries[5m]) > 10
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High rate of slow queries detected"
```

---

## 安全加固

### 1. 用户权限管理

```sql
-- 创建应用用户（最小权限原则）
CREATE USER 'app_user'@'%' IDENTIFIED BY 'StrongPassword123!';
GRANT SELECT, INSERT, UPDATE, DELETE ON your_database.* TO 'app_user'@'%';
FLUSH PRIVILEGES;

-- 创建只读用户
CREATE USER 'readonly_user'@'%' IDENTIFIED BY 'ReadOnlyPass456!';
GRANT SELECT ON your_database.* TO 'readonly_user'@'%';
FLUSH PRIVILESES;

-- 撤销不必要的权限
REVOKE DROP, ALTER, CREATE ON your_database.* FROM 'app_user'@'%';

-- 查看用户权限
SHOW GRANTS FOR 'app_user'@'%';
```

### 2. 密码策略

```sql
-- 启用密码强度插件
INSTALL PLUGIN validate_password SONAME 'validate_password.so';

-- 配置密码策略
SET GLOBAL validate_password.policy = MEDIUM;
SET GLOBAL validate_password.length = 12;
SET GLOBAL validate_password.mixed_case_count = 2;
SET GLOBAL validate_password.number_count = 2;
SET GLOBAL validate_password.special_char_count = 2;

-- 强制密码过期
ALTER USER 'app_user'@'%' PASSWORD EXPIRE INTERVAL 90 DAY;
```

### 3. SSL/TLS 加密

```sql
-- 生成 SSL 证书（生产环境应使用 CA 签发的证书）
openssl req -newkey rsa:2048 -days 365 -nodes -keyout server-key.pem -out server-cert.pem

-- 配置 MySQL 使用 SSL
[mysqld]
ssl-ca = C:/MySQL/SSL/ca-cert.pem
ssl-cert = C:/MySQL/SSL/server-cert.pem
ssl-key = C:/MySQL/SSL/server-key.pem

-- 要求连接使用 SSL
ALTER USER 'app_user'@'%' REQUIRE SSL;

-- 测试 SSL 连接
mysql --ssl-mode=REQUIRED -u app_user -p -h your_server
```

### 4. 防火墙配置

```powershell
# Windows 防火墙规则
# 仅允许特定 IP 访问 MySQL 端口

# 添加入站规则
New-NetFirewallRule -DisplayName "MySQL Access" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 3306 `
    -RemoteAddress "192.168.1.0/24" `
    -Action Allow

# 删除默认允许所有 IP 的规则
Remove-NetFirewallRule -DisplayName "MySQL All"
```

### 5. 审计日志

```sql
-- 启用审计日志插件
INSTALL PLUGIN audit_log SONAME 'audit_log.so';

-- 配置审计策略
SET GLOBAL audit_log_policy = 'ALL';
SET GLOBAL audit_log_include_users = 'app_user,admin_user';

-- 查看审计日志
SELECT * FROM mysql.audit_log WHERE event_type = 'CONNECT';
```

---

## 高可用方案

### 1. MySQL 主从复制

```ini
# 主服务器配置 (my.ini)
[mysqld]
server-id = 1
log_bin = D:/MySQL/BinLog/mysql-bin
binlog_format = ROW
binlog_do_db = your_database
expire_logs_days = 7
max_connections = 1000
```

```ini
# 从服务器配置 (my.ini)
[mysqld]
server-id = 2
log_bin = D:/MySQL/BinLog/mysql-bin
relay_log = D:/MySQL/RelayLog/relay-bin
read_only = ON
super_read_only = ON
```

```sql
-- 在主服务器上创建复制用户
CREATE USER 'repl_user'@'%' IDENTIFIED BY 'ReplPassword789!';
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%';
FLUSH PRIVILEGES;

-- 获取主服务器状态
SHOW MASTER STATUS;

-- 在从服务器上配置复制
CHANGE MASTER TO
    MASTER_HOST='primary_server_ip',
    MASTER_USER='repl_user',
    MASTER_PASSWORD='ReplPassword789!',
    MASTER_LOG_FILE='mysql-bin.000001',
    MASTER_LOG_POS=154;

-- 启动复制
START SLAVE;

-- 检查复制状态
SHOW SLAVE STATUS\G
```

### 2. MySQL Group Replication

```ini
# Group Replication 配置
[mysqld]
plugin_load_add = 'group_replication.so'
group_replication_group_name = "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"
group_replication_local_address = "server1:33061"
group_replication_group_seeds = "server1:33061,server2:33061,server3:33061"
group_replication_bootstrap_group = OFF
```

### 3. MySQL InnoDB Cluster

```javascript
// MySQL Shell 配置 InnoDB Cluster
var cluster = dba.createCluster('MyCluster');
cluster.addInstance('root@server2:3306');
cluster.addInstance('root@server3:3306');
cluster.status();

// 创建路由用户
cluster.createRouterAccount('router_user', 'RouterPassword123!');
```

### 4. 负载均衡

```nginx
# Nginx 反向代理配置
stream {
    upstream mysql_backend {
        server primary_server:3306 weight=3;
        server replica1:3306 weight=1;
        server replica2:3306 weight=1;
    }
    
    server {
        listen 3306;
        proxy_pass mysql_backend;
        proxy_timeout 3s;
        proxy_connect_timeout 5s;
    }
}
```

---

## 日常维护

### 1. 每日检查清单

```sql
-- 1. 检查错误日志
-- Windows: 查看 Event Viewer -> Application -> MySQL

-- 2. 检查复制状态
SHOW SLAVE STATUS\G

-- 3. 检查磁盘空间
SELECT 
    table_schema AS database_name,
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS size_mb
FROM information_schema.tables
GROUP BY table_schema
ORDER BY size_mb DESC;

-- 4. 检查慢查询
SELECT COUNT(*) FROM mysql.slow_log;

-- 5. 检查锁等待
SELECT * FROM performance_schema.data_lock_waits;

-- 6. 检查连接数
SHOW STATUS LIKE 'Threads_connected';
SHOW VARIABLES LIKE 'max_connections';
```

### 2. 每周维护任务

```sql
-- 1. 优化表
OPTIMIZE TABLE your_database.your_table;

-- 2. 分析表
ANALYZE TABLE your_database.your_table;

-- 3. 检查碎片
SELECT 
    table_name,
    data_free,
    ROUND(data_free / 1024 / 1024, 2) AS free_mb
FROM information_schema.tables
WHERE data_free > 0
ORDER BY data_free DESC;

-- 4. 清理二进制日志
PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 7 DAY);

-- 5. 检查备份完整性
-- 手动测试备份恢复
```

### 3. 每月维护任务

```sql
-- 1. 更新统计信息
UPDATE STATISTICS;

-- 2. 检查用户权限
SELECT user, host, password_expired, password_last_changed 
FROM mysql.user;

-- 3. 审查审计日志
SELECT * FROM mysql.audit_log 
WHERE event_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
AND event_type IN ('FAILED_LOGIN', 'ACCOUNT_LOCKED');

-- 4. 性能趋势分析
-- 对比历史性能数据
```

### 4. 补丁与升级

```powershell
# 升级前检查
# 1. 确认当前版本
SELECT VERSION();

# 2. 检查兼容性
# 参考 MySQL 官方升级指南

# 3. 备份所有数据
# 执行完整备份

# 4. 在测试环境验证
# 先在非生产环境测试升级

# 5. 计划维护窗口
# 通知相关人员
```

---

## 故障排查

### 1. 常见问题及解决方案

#### 问题 1：MySQL 无法启动

```powershell
# 检查错误日志
Get-Content "D:\MySQL\Logs\mysql-error.log" -Tail 50

# 检查端口占用
netstat -ano | findstr :3306

# 检查服务状态
Get-Service MySQL80

# 重新初始化（最后手段）
mysqld --initialize-insecure
```

#### 问题 2：连接数过多

```sql
-- 查看当前连接
SHOW PROCESSLIST;

-- 查看连接分布
SELECT 
    HOST,
    DB,
    COUNT(*) AS connections
FROM information_schema.processes
GROUP BY HOST, DB
ORDER BY connections DESC;

-- 终止异常连接
KILL connection_id;

-- 临时增加连接数
SET GLOBAL max_connections = 1000;
```

#### 问题 3：磁盘空间不足

```sql
-- 检查各数据库大小
SELECT 
    table_schema AS database_name,
    ROUND(SUM(data_length) / 1024 / 1024, 2) AS data_size_mb,
    ROUND(SUM(index_length) / 1024 / 1024, 2) AS index_size_mb
FROM information_schema.tables
GROUP BY table_schema
ORDER BY SUM(data_length) DESC;

-- 清理大表
DELETE FROM your_table WHERE created_at < DATE_SUB(NOW(), INTERVAL 90 DAY);

-- 收缩表
OPTIMIZE TABLE your_table;
```

#### 问题 4：性能下降

```sql
-- 1. 检查活跃查询
SELECT 
    id,
    user,
    host,
    db,
    command,
    time,
    state,
    info
FROM information_schema.processes
WHERE command != 'Sleep'
ORDER BY time DESC;

-- 2. 检查缓冲池命中率
SHOW STATUS LIKE 'Innodb_buffer_pool_read%';

-- 3. 检查锁等待
SELECT * FROM performance_schema.data_lock_waits;

-- 4. 检查 IO 负载
SHOW ENGINE INNODB STATUS\G
```

### 2. 使用 Performance Schema 诊断

```sql
-- 查看最耗资源的语句
SELECT 
    DIGEST_TEXT,
    COUNT_STAR,
    ROUND(AVG_TIMER_WAIT / 1000000000, 2) AS avg_latency_ms,
    ROUND(SUM_TIMER_WAIT / 1000000000, 2) AS total_latency_s
FROM performance_schema.events_statements_summary_by_digest
ORDER BY total_latency_s DESC
LIMIT 10;

-- 查看最耗资源的表
SELECT 
    OBJECT_SCHEMA,
    OBJECT_NAME,
    COUNT_READ,
    COUNT_WRITE,
    COUNT_FETCH
FROM performance_schema.table_io_waits_summary_by_table
ORDER BY (COUNT_READ + COUNT_WRITE + COUNT_FETCH) DESC
LIMIT 10;
```

### 3. Windows 事件日志排查

```powershell
# 查看 MySQL 相关事件
Get-EventLog -LogName Application -Source MySQL -Newest 20

# 查看系统资源问题
Get-EventLog -LogName System -EntryType Error -Newest 10

# 查看磁盘错误
Get-EventLog -LogName System -Source disk -Newest 10
```

---

## 总结

在 Windows 平台上运维 MySQL 生产环境，需要关注以下几个核心方面：

1. **合理配置**：根据硬件资源和业务需求调整参数
2. **持续监控**：建立完善的监控和告警体系
3. **定期备份**：确保数据安全，定期测试恢复流程
4. **安全防护**：实施最小权限原则，启用加密和审计
5. **高可用设计**：根据业务连续性要求选择合适的方案
6. **日常维护**：制定并执行标准化的维护计划
7. **故障应急**：建立应急预案，定期进行演练

> **最后提醒**：任何变更都应在测试环境充分验证后再应用到生产环境。保持详细的操作记录，便于问题追溯和经验积累。

---

## 参考资料

- [MySQL 官方文档](https://dev.mysql.com/doc/)
- [MySQL 性能优化指南](https://dev.mysql.com/doc/refman/8.0/en/performance.html)
- [Windows Server 最佳实践](https://docs.microsoft.com/windows-server/)
- [Percona 工具集](https://www.percona.com/software)
- [MySQL 高可用方案比较](https://dev.mysql.com/doc/refman/8.0/en/replication.html)

---

*本文档最后更新时间：2026-10-01*
*作者：VeryCAD Team*