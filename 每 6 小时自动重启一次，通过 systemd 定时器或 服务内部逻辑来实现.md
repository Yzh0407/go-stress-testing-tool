# **每 6 小时自动重启一次**，通过 **systemd 定时器**或 **服务内部逻辑**来实现

---

## **方法 1：使用 systemd 定时器（推荐）**
### **步骤 1：修改服务文件**
确保服务文件（`/etc/systemd/system/go-stress-test.service`）中没有 `Restart=always`，因为我们要手动控制重启：
```ini
[Unit]
Description=High-Concurrency Stress Test Service
After=network.target

[Service]
Type=simple
User=root
ExecStart=/root/go/bin/go-stress-testing -c 10000 -n 10000 -u https://www.wssll.cn:443
Restart=no
LimitNOFILE=100000
Environment="GODEBUG=netdns=go"

[Install]
WantedBy=multi-user.target
```

### **步骤 2：创建定时器文件**
```bash
sudo nano /etc/systemd/system/go-stress-test.timer
```
粘贴以下内容：
```ini
[Unit]
Description=Timer to restart go-stress-test every 6 hours

[Timer]
OnCalendar=*-*-* 0/6:00:00
Persistent=true
Unit=go-stress-test.service

[Install]
WantedBy=timers.target
```

### **步骤 3：启用定时器**
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now go-stress-test.timer
```

### **验证定时器**
```bash
sudo systemctl list-timers
```
• **输出示例**：
  ```
  NEXT                         LEFT       LAST                         PASSED    UNIT                     ACTIVATES
  Fri 2024-03-15 18:00:00 UTC  5h left    Fri 2024-03-15 12:00:00 UTC  1h ago    go-stress-test.timer     go-stress-test.service
  ```

---

## **方法 2：服务内部逻辑（简单但不够灵活）**
如果不想用定时器，可以在服务文件中添加一个脚本逻辑，每 6 小时重启一次：

### **修改服务文件**
```bash
sudo nano /etc/systemd/system/go-stress-test.service
```
粘贴以下内容：
```ini
[Unit]
Description=High-Concurrency Stress Test Service
After=network.target

[Service]
Type=simple
User=root
ExecStart=/bin/bash -c 'while true; do /root/go/bin/go-stress-testing -c 10000 -n 10000 -u https://www.wssll.cn:443; sleep 21600; done'  # 21600 秒 = 6 小时
Restart=always
LimitNOFILE=100000
Environment="GODEBUG=netdns=go"

[Install]
WantedBy=multi-user.target
```

### **重启服务**
```bash
sudo systemctl daemon-reload
sudo systemctl restart go-stress-test
```

---

## **方法对比**
| **方法**           | **优点**               | **缺点**                     |
| ------------------ | ---------------------- | ---------------------------- |
| **systemd 定时器** | 灵活、可扩展、易于管理 | 需要额外配置定时器文件       |
| **服务内部逻辑**   | 简单、无需额外文件     | 不够灵活、重启逻辑与服务耦合 |
