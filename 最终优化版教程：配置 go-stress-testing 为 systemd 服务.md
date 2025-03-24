## **最终优化版教程：配置 go-stress-testing 为 systemd 服务**

### **1. 确认 go-stress-testing 安装**
运行以下命令，确认 `go-stress-testing` 已安装并获取路径：
```bash
ls -l /root/go/bin/go-stress-testing
```
• **正常输出**：
  ```
  -rwxr-xr-x 1 root root 10M May 20 10:00 /root/go/bin/go-stress-testing
  ```
  如果文件不存在或没有执行权限，运行：
  ```bash
  chmod +x /root/go/bin/go-stress-testing
  ```

---

### **2. 创建 systemd 服务文件**
```bash
sudo nano /etc/systemd/system/go-stress-test.service
```
粘贴以下内容（**已适配你的路径和参数**）：
```ini
[Unit]
Description=High-Concurrency Stress Test Service
After=network.target

[Service]
Type=simple
User=0  # 使用 root 用户（UID=0）
ExecStart=/root/go/bin/go-stress-testing -c 10000 -n 10000 -u https://www.wssll.cn:443
Restart=always  # 崩溃后自动重启
RestartSec=5    # 重启间隔（秒）
LimitNOFILE=100000  # 提高文件描述符限制（应对高并发）
Environment="GODEBUG=netdns=go"  # 强制使用 Go 的 DNS 解析器

[Install]
WantedBy=multi-user.target  # 开机自启
```

---

### **3. 生效并启动服务**
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now go-stress-test
```

---

### **4. 验证服务状态**
```bash
sudo systemctl status go-stress-test
```
• **正常输出**：
  ```
  ● go-stress-test.service - High-Concurrency Stress Test Service
     Loaded: loaded (/etc/systemd/system/go-stress-test.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2024-03-15 10:00:00 UTC; 10s ago
   Main PID: 1234 (go-stress-testi)
      Tasks: 10 (limit: 4915)
     Memory: 50.0M
     CGroup: /system.slice/go-stress-test.service
             └─1234 /root/go/bin/go-stress-testing -c 10000 -n 10000 -u https://www.wssll.cn:443
  ```

---

### **5. 查看实时日志**
```bash
sudo journalctl -u go-stress-test -f
```
• **正常输出**：
  ```
  Mar 15 10:00:05 s18900 go-stress-testing[1234] 开始压力测试...
  Mar 15 10:00:05 s18900 go-stress-testing[1234] 并发数:10000 总请求数:10000
  Mar 15 10:00:08 s18900 go-stress-testing[1234] 成功请求:1000 失败:0
  ```

---

### **6. 常见问题排查**
#### **问题 1：服务无法启动**
```bash
# 查看详细日志
journalctl -u go-stress-test -xe
```
• **可能原因**：
  • `ExecStart` 路径错误（检查 `/root/go/bin/go-stress-testing` 是否存在）
  • 文件描述符限制不足（确保 `LimitNOFILE=100000` 已生效）

#### **问题 2：DNS 解析失败**
```log
dial tcp: lookup www.wssll.cn:443: no such host
```
**修复**：
```bash
# 修改服务文件，强制使用 Go 的 DNS 解析器
Environment="GODEBUG=netdns=go"
sudo systemctl restart go-stress-test
```

#### **问题 3：端口耗尽**
```log
dial tcp: lookup www.wssll.cn:443: socket: too many open files
```
**修复**：
```bash
# 检查文件描述符限制
cat /proc/$(pgrep go-stress-testing)/limits | grep 'Max open files'
# 如果未生效，重启服务
sudo systemctl restart go-stress-test
```

---

### **7. 管理服务**
#### **启动服务**
```bash
sudo systemctl start go-stress-test
```

#### **停止服务**
```bash
sudo systemctl stop go-stress-test
```

#### **重启服务**
```bash
sudo systemctl restart go-stress-test
```

#### **禁用开机自启**
```bash
sudo systemctl disable go-stress-test
```

---

#### **启动运行服务和自动重启服务**
```bash
sudo systemctl start go-stress-test.timer
```
```bash
sudo systemctl start go-stress-test.service
```

#### **停止运行服务和自动重启服务**
```bash
sudo systemctl stop go-stress-test.timer
```
```bash
sudo systemctl stop go-stress-test.service
```

#### **运行服务和自动重启服务状态检查**
```bash
sudo systemctl status go-stress-test.timer
```
```bash
sudo systemctl status go-stress-test.service
```




### **8. 总结**
通过本教程，你已经将 `go-stress-testing` 配置为一个 **持久化运行的 systemd 服务**，具备以下特性：
• **断线无视**：即使 SSH 断开，服务仍继续运行。
• **崩溃自动重启**：如果程序崩溃或被杀死，服务会自动重启。
• **高并发支持**：通过 `LimitNOFILE` 提升文件描述符限制，支持高并发测试。
