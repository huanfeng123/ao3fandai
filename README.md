# AO3 反向代理与动态 IP 切换系统

## 📖 项目简介

这是一个为 [Archive of Our Own (AO3)](https://archiveofourown.org) 设计的反向代理系统，集成了动态 IP 切换功能。核心功能包括：
- **智能反向代理**：通过 Docker 容器运行，提供安全的 AO3 访问代理
- **动态 IP 池**：利用 Mihomo 代理池，每 10 分钟自动切换出口 IP，有效避免访问频率限制
- **Cookie 管理**：支持用户 Cookie 上传和自动重写，维持登录状态
- **域名重写**：自动替换页面内的 AO3 域名，提供无缝的代理体验

---

##  演示地址
ao3镜像[https://www.ao3.icu/](https://www.ao3.icu/)

## 🏗️ 系统架构

```
用户请求 → Docker反向代理(8000端口) → Mihomo代理池(7890端口) → AO3源站
        ↑                                ↑
    Cookie管理                       动态IP切换
```

## 📁 项目结构

```
├── proxy-pool/                    # Mihomo 代理池核心
│   ├── mihomo                     # Mihomo 主程序 (需自行下载对应平台版本)
│   ├── mihomo-config.yaml         # 代理池配置文件
│   ├── start.sh                   # 一键启动脚本
│   ├── stop.sh                    # 停止服务脚本
│   ├── switch.sh                  # 手动切换 IP 脚本
│   ├── status.sh                  # 服务状态检查脚本
│   ├── providers/                 # 代理订阅缓存目录 (自动生成)
│   └── logs/                      # 运行日志目录 (自动生成)
│
└── ao3-proxy/                     # Docker 反向代理服务
    ├── docker-compose.yml         # Docker 服务编排配置
    ├── Dockerfile                 # Deno 应用容器构建文件
    ├── deno.json                  # Deno 运行时配置
    ├── main.ts                    # 反向代理核心逻辑 (TypeScript)
    ├── start.sh                   # 项目启动脚本
    └── README.md                  # 本文件
```

---

## ⚙️ 核心配置详解

### 1. Mihomo 代理池配置 (`proxy-pool/mihomo-config.yaml`)

这是代理池的核心配置文件，**部署时必须检查以下关键项**：

```yaml
# 网络监听配置
mixed-port: 7890                    # HTTP/SOCKS5 混合代理端口
allow-lan: true                     # ✅ 必须设为 true，允许 Docker 容器访问
external-controller: 0.0.0.0:9090   # ✅ 必须监听 0.0.0.0，API 才能正常工作
external-ui: dashboard              # Web 控制面板

# 代理订阅源（需要定期更新）
proxy-providers:
  provider1:
    type: http
    url: "YOUR_SUBSCRIPTION_URL_1"  # 替换为你的实际订阅链接
    interval: 3600
  provider2:
    type: http
    url: "YOUR_SUBSCRIPTION_URL_2"  # 替换为你的实际订阅链接
    interval: 3600

# 代理策略
proxy-groups:
  - name: 自动切换
    type: select                    # 选择器模式，用于手动/定时切换
    use:
      - provider1
      - provider2

# 路由规则
rules:
  - MATCH,自动切换                 # 所有流量走“自动切换”代理组
```

**⚠️ 重要提醒**：
- `allow-lan: true` 是 Docker 容器能访问代理服务的关键
- `external-controller` 必须设置为 `0.0.0.0:9090`，否则 API 无法工作
- 订阅链接通常有有效期，请及时更新

### 2. Docker 反向代理配置 (`ao3-proxy/docker-compose.yml`)

```yaml
services:
  app:
    build: .
    environment:
      PORT: "8000"                           # 服务监听端口
      PROXY_HOST: "your-domain.com"          # ⚠️ 替换为你的访问域名
      HTTP_PROXY: "http://宿主机内网IP:7890"  # ⚠️ 必须修改为服务器内网IP
      HTTPS_PROXY: "http://宿主机内网IP:7890" # ⚠️ 同上
    ports:
      - "8000:8000"                          # 端口映射：主机8000 → 容器8000
    restart: unless-stopped
```

---

## 🚀 快速部署指南

### 前提条件
- Linux 服务器 (Ubuntu/CentOS 等)
- Docker 和 Docker Compose 已安装
- 一个可用的域名 (用于 `PROXY_HOST`)
- 可用的代理订阅链接 (用于 `mihomo-config.yaml`)

### 部署步骤

#### 第 1 步：获取服务器内网 IP
在部署前，需要获取服务器的内网 IP 地址：

```bash
# 在新服务器上执行
hostname -I | awk '{print $1}'
# 示例输出：10.0.2.18 或 192.168.1.100
```

**记下这个 IP**，下一步配置 Docker 时需要用到。

#### 第 2 步：修改 Docker 配置中的代理地址
**这是最常见的部署错误来源！**

编辑 `ao3-proxy/docker-compose.yml` 文件，将 `HTTP_PROXY` 和 `HTTPS_PROXY` 环境变量值中的 **`宿主机内网IP`** 替换为你在第 1 步获取的实际内网 IP。

```yaml
# 修改前
HTTP_PROXY: "http://宿主机内网IP:7890"

# 修改后（假设你的内网IP是 10.0.2.18）
HTTP_PROXY: "http://10.0.2.18:7890"
```

#### 第 3 步：配置 main.ts 中的代理域名
在 `ao3-proxy/main.ts` 文件中，找到以下配置行：

```typescript
const PROXY_HOST = Deno.env.get("PROXY_HOST") || "www.ao3.icu";
```

确保此处的默认域名与你在 `docker-compose.yml` 中设置的 `PROXY_HOST` 一致，或者通过环境变量传入正确的域名。

#### 第 4 步：启动所有服务

```bash
# 1. 启动 Mihomo 代理池
cd proxy-pool
chmod +x *.sh mihomo     # 确保脚本和程序有执行权限
./start.sh               # 一键启动代理池和定时任务

# 2. 启动 Docker 反向代理
cd ../ao3-proxy
docker-compose build     # 首次需要构建镜像
docker-compose up -d     # 后台启动服务
```

#### 第 5 步：测试访问

```bash
# 测试本地访问（应该返回AO3页面或代理响应）
curl -v http://localhost:8000/

# 测试代理池工作（每次运行可能显示不同IP）
curl -x http://localhost:7890 https://api.ipify.org

# 查看服务状态
cd ../proxy-pool && ./status.sh
cd ../ao3-proxy && docker-compose ps
```

---

## 🔧 日常使用与管理

### 服务管理命令

```bash
# Mihomo 代理池管理
cd proxy-pool
./start.sh      # 启动服务
./stop.sh       # 停止服务
./status.sh     # 查看状态和当前IP
./switch.sh     # 手动立即切换IP

# Docker 服务管理
cd ao3-proxy
docker-compose up -d      # 启动服务
docker-compose down       # 停止服务
docker-compose logs -f    # 查看实时日志
docker-compose ps         # 查看容器状态
```

### IP 切换功能
- **自动切换**：已配置为每 10 分钟自动切换一次 IP
- **手动切换**：可随时执行 `./switch.sh` 强制切换
- **切换验证**：切换后访问 `http://你的域名:8000/proxy-test` 可查看当前 IP

### Cookie 管理
1. 访问 `http://你的域名:8000/cookie`
2. 粘贴从 AO3 获取的 Cookie 字符串
3. 点击保存，系统将自动重写 Cookie 域名
4. 之后访问即可保持登录状态

---

## ⚠️ 故障排查指南

### 问题 1：端口未监听
**症状**：`curl http://localhost:8000` 无响应或连接拒绝

**诊断与解决：**
```bash
# 检查端口监听状态
ss -tulpn | grep -E '(7890|8000|9090)'

# 如果没有输出，说明服务未启动
# 检查 Mihomo 进程
ps aux | grep mihomo | grep -v grep

# 检查 Docker 容器
docker-compose ps

# 常见原因及解决：
# 1. Mihomo未启动 → cd proxy-pool && ./start.sh
# 2. Docker容器未运行 → cd ao3-proxy && docker-compose up -d
# 3. 配置文件错误 → 检查 mihomo-config.yaml 语法
```

### 问题 2：防火墙阻止访问
**症状**：服务器本地可访问，但外部无法连接

**诊断与解决：**
```bash
# 检查防火墙状态
sudo ufw status                  # Ubuntu/Debian
sudo firewall-cmd --state        # CentOS/RHEL
sudo iptables -L -n              # 查看所有规则

# 临时禁用防火墙测试（仅用于诊断）
sudo ufw disable
# 或
sudo systemctl stop firewalld

# 永久解决方案：开放必要端口
sudo ufw allow 8000/tcp          # 反向代理端口
sudo ufw allow 7890/tcp          # 代理池端口
sudo ufw allow 9090/tcp          # 管理API端口
sudo ufw reload
```

### 问题 3：容器无法连接代理
**症状**：Docker 日志显示 "Connection refused" 或 "tcp connect error"

**诊断与解决：**
```bash
# 进入容器内部诊断
docker-compose exec app sh

# 在容器内测试
echo "HTTP_PROXY: $HTTP_PROXY"
ping <宿主机IP>                  # 测试网络连通性
curl -v $HTTP_PROXY              # 测试代理连接

# 常见原因：
# 1. docker-compose.yml 中的 HTTP_PROXY IP 错误 → 修改为正确的内网IP
# 2. Mihomo 的 allow-lan 未设置为 true → 检查 mihomo-config.yaml
# 3. 宿主机防火墙阻止了 7890 端口 → 见问题2的防火墙解决方案
```

### 问题 4：代理订阅失效
**症状**：无法访问外部网站，或 IP 切换失败

**诊断与解决：**
```bash
# 检查代理池节点状态
curl http://localhost:9090/proxies | python3 -m json.tool

# 查看 Mihomo 日志
tail -f proxy-pool/mihomo.log

# 解决方案：
# 1. 更新 mihomo-config.yaml 中的订阅链接
# 2. 清理缓存：rm -rf proxy-pool/providers/*.yaml
# 3. 重启服务：./stop.sh && ./start.sh
```

### 问题 5：域名重写不正确
**症状**：页面链接仍指向 AO3 原域名或跳转到错误域名

**诊断与解决：**
```bash
# 检查环境变量
docker-compose exec app sh -c 'echo "PROXY_HOST: $PROXY_HOST"'

# 查看请求日志
docker-compose logs --tail=50 app

# 解决方案：
# 1. 确保 docker-compose.yml 中的 PROXY_HOST 设置正确
# 2. 检查 main.ts 中的 PROXY_HOST 默认值
# 3. 清理浏览器缓存后重试
```

---

## 🔄 维护与监控

### 日志管理
```bash
# Mihomo 日志
tail -f proxy-pool/mihomo.log
tail -f proxy-pool/logs/switch_*.log

# Docker 应用日志
docker-compose logs -f --tail=50

# 查看错误日志
grep -i "error\|fail" proxy-pool/mihomo.log
docker-compose logs | grep -i "error"
```

### 性能监控
```bash
# 监控 Docker 资源使用
docker stats

# 监控服务器资源
htop
df -h
free -h
```

### 备份与恢复
```bash
# 备份配置文件
tar -czf ao3-proxy-backup-$(date +%Y%m%d).tar.gz \
  --exclude='proxy-pool/providers/*.yaml' \
  --exclude='*.log' \
  proxy-pool/ ao3-proxy/

# 从备份恢复
tar -xzf ao3-proxy-backup-YYYYMMDD.tar.gz
cd proxy-pool && ./start.sh
cd ../ao3-proxy && docker-compose up -d
```

---

## 📋 部署检查清单

在部署完成后，请逐一验证：

- [ ] **网络配置**
  - [ ] 服务器内网 IP 已正确配置到 `docker-compose.yml`
  - [ ] 防火墙已开放 8000、7890、9090 端口
  - [ ] 域名 DNS 解析已指向服务器公网 IP

- [ ] **服务状态**
  - [ ] Mihomo 代理池正常运行 (`./status.sh`)
  - [ ] Docker 容器状态正常 (`docker-compose ps`)
  - [ ] 本地访问测试通过 (`curl http://localhost:8000`)
  - [ ] 代理功能测试通过 (`curl -x http://localhost:7890 https://api.ipify.org`)

- [ ] **功能验证**
  - [ ] 通过域名可正常访问 AO3 页面
  - [ ] 页面内所有链接已正确重写
  - [ ] Cookie 上传功能正常工作
  - [ ] IP 切换功能正常（每 10 分钟自动切换）

- [ ] **安全配置**
  - [ ] 订阅链接已更新为自己的有效链接
  - [ ] 考虑设置 API 访问密码 (`secret: "your-password"`)
  - [ ] 定期检查日志，排查异常访问

---

## 🆘 紧急恢复流程

如果服务完全不可用，按顺序执行：

1. **检查基础服务**
   ```bash
   cd proxy-pool && ./status.sh
   cd ../ao3-proxy && docker-compose ps
   ```

2. **查看错误信息**
   ```bash
   tail -100 proxy-pool/mihomo.log
   docker-compose logs --tail=100
   ```

3. **重启服务**
   ```bash
   cd proxy-pool && ./stop.sh && ./start.sh
   cd ../ao3-proxy && docker-compose restart
   ```

4. **检查网络**
   ```bash
   ping 8.8.8.8
   curl -v https://google.com
   ss -tulpn | grep -E '(7890|8000)'
   ```

5. **回滚配置**
   如果最近修改过配置文件，恢复到最后一次正常工作的版本。

---

## 获取帮助

遇到问题时，请按以下顺序排查：
1. 查看本 README 的故障排查章节
2. 检查日志文件中的错误信息
3. 验证配置文件的关键设置
4. 确认网络和防火墙配置

如果仍无法解决，请提供以下信息：
- 错误日志片段
- 相关配置文件内容（隐藏敏感信息）
- 执行 `./status.sh` 的输出
- 执行 `docker-compose ps` 的输出

---

## 📄 许可证与免责声明

本项目仅供学习和研究使用，请遵守：
1. AO3 的服务条款和 robots.txt
2. 当地法律法规
3. 代理服务提供商的使用条款

使用者需自行承担因使用本项目产生的所有责任和风险。

**最后更新**：2025年12月

**保持更新**：请定期检查项目更新，获取最新的功能和安全修复。
