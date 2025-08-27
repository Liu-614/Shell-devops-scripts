```markdown
# Shell 运维工具集
包含系统监控和日志清理两个脚本，适用于 Linux 服务器日常运维。
## 功能
### 系统监控（system_monitor.sh）
- 实时监控 CPU、内存、磁盘使用率
- 超过阈值时发送告警邮件，并记录日志
- 支持通过 config.conf 自定义阈值和邮件配置
### 日志清理（log_cleaner.sh）
- 定期清理指定目录下超过 7 天的日志文件
- 支持自定义日志路径和保留天数
- 清理结果记录到日志文件
## 文件结构
```
shell-sysadmin-tools/
├── README.md               # 项目说明文档
├── system_monitor.sh       # 系统监控脚本
├── log_cleaner.sh          # 日志清理脚本
└── config.conf             # 配置文件（阈值/邮件/日志路径）

```
## 使用方法
1. 修改配置文件 `config.conf`（设置阈值、邮件、日志路径等）
2. 赋予脚本执行权限：`chmod +x system_monitor.sh log_cleaner.sh`
3. 手动测试：
   - 监控：`./system_monitor.sh`
   - 清理：`./log_cleaner.sh`
4. 设置定时任务（如每天凌晨监控和清理）：
   ```bash
   crontab -e
   # 添加以下内容（每天 1 点执行监控，2 点执行清理）
   0 1 * * * /path/to/system_monitor.sh
   0 2 * * * /path/to/log_cleaner.sh
```
## 配置文件示例（config.conf）
```bash
# 系统监控阈值（单位：%）
CPU_THRESHOLD=80
MEMORY_THRESHOLD=85
DISK_THRESHOLD=90
# 告警邮件配置
EMAIL_TO="admin@example.com"
EMAIL_SUBJECT="系统资源告警"
# 日志清理配置
LOG_DIR="/var/log"
LOG_DAYS=7
```
## 代码实现
### system_monitor.sh（系统监控脚本）
```bash
#!/bin/bash
# 读取配置文件
source config.conf
# 获取系统资源使用率
cpu_usage=$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 -$1}')
memory_usage=$(free -m | awk 'NR==2{printf "%.2f",$3*100/$2}')
disk_usage=$(df -h | awk '$NF=="/"{printf "%s", $5}' | sed 's/%//')
# 检查是否超过阈值并发送告警
send_alert() {
    local resource=$1
    local usage=$2
    local message="警告：服务器 ${resource} 使用率过高！当前使用率:${usage}%"
    echo "${message}" | mail -s "${EMAIL_SUBJECT}" "${EMAIL_TO}"
    echo "$(date '+%Y-%m-%d %H:%M:%S') -${message}" >> /var/log/system_monitor.log
}
# 判断 CPU
if (( $(echo "${cpu_usage} >= ${CPU_THRESHOLD}" | bc -l) )); then
    send_alert "CPU" "${cpu_usage}"
fi
# 判断内存
if (( $(echo "${memory_usage} >= ${MEMORY_THRESHOLD}" | bc -l) )); then
    send_alert "内存" "${memory_usage}"
fi
# 判断磁盘
if [ "${disk_usage}" -ge "${DISK_THRESHOLD}" ]; then
    send_alert "磁盘" "${disk_usage}"
fi
echo "$(date '+%Y-%m-%d %H:%M:%S') - 监控完成，CPU:${cpu_usage}%, 内存: ${memory_usage}%, 磁盘:${disk_usage}%" >> /var/log/system_monitor.log
```
### log_cleaner.sh（日志清理脚本）
```bash
#!/bin/bash
# 读取配置文件
source config.conf
# 清理超过指定天数的日志文件
clean_logs() {
    local log_dir=$1
    local days=$2
    if [ ! -d "${log_dir}" ]; then
        echo "错误：日志目录 ${log_dir} 不存在！"
        return 1
    fi
    find "${log_dir}" -type f -name "*.log" -mtime +${days} -exec rm -f {} \;
    echo "$(date '+%Y-%m-%d %H:%M:%S') - 已清理${log_dir} 下超过 ${days} 天的日志文件" >> /var/log/log_cleaner.log
}
# 执行清理
clean_logs "${LOG_DIR}" "${LOG_DAYS}"
```
## 注意事项
1. 需提前安装邮件服务（如 sendmail 或 postfix）才能发送告警邮件
2. 日志清理操作不可逆，建议先备份重要日志