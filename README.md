# AO3 反向代理与动态IP切换系统部署指南

## 📦 项目概述
本系统由两个核心部分组成：
1. **Mihomo 代理池** - 提供动态IP切换功能，每10分钟自动更换出口IP
2. **Docker 反向代理** - 基于Deno的AO3反向代理服务，通过代理池访问源站

## 🗂️ 项目结构
```
├── proxy-pool/                    # Mihomo代理池
│   ├── mihomo                     # 主程序
│   ├── mihomo-config.yaml         # 核心配置文件
│   ├── start.sh                   # 启动脚本
│   ├── stop.sh                    # 停止脚本
│   ├── switch.sh                  # 手动切换IP脚本
│   └── providers/                 # 订阅缓存目录
│
└── ao3-proxy/                     # Docker反向代理
    ├── docker-compose.yml         # Docker编排配置
    ├── Dockerfile                 # 容器构建文件
    ├── deno.json                  # Deno任务配置
    ├── main.ts                    # 反向代理主程序
    └── start.sh                   # 一键启动脚本
```

## 🔧 核心配置文件详解

### 1. Mihomo 代理池配置 (`/root/proxy-pool/mihomo-config.yaml`)
**关键配置项：**
```yaml
mixed-port: 7890                    # HTTP/SOCKS5混合代理端口
allow-lan: true                     # ✅ 必须为true，允许外部访问
external-controller: 0.0.0.0:9090   # ✅ API服务地址（必须监听所有IP）
```

**注意事项：**
- `allow-lan: true` 确保代理可被Docker容器访问
- `external-controller: 0.0.0.0:9090` 是API正常工作前提
- 代理订阅链接需定期更新，免费订阅通常7-30天有效期

### 2. Docker 反向代理配置 (`/www/docker/ao3-proxy/`)
**docker-compose.yml 关键配置：**
```yaml
environment:
  PORT: "8000"                     # 服务端口
  PROXY_HOST: "your-domain.com"    # 你的代理域名
  HTTP_PROXY: "http://宿主机IP:7890" # 指向Mihomo代理
```

**环境变量说明：**
- `PROXY_HOST`: 设置你的访问域名（如 `www.ao3.icu`）
- `HTTP_PROXY`: **必须修改为宿主机内网IP**，容器通过此地址访问代理

## 🚀 部署步骤

### 第1步：获取服务器内网IP
在新服务器上执行：
```bash
hostname -I | awk '{print $1}'
```
记录输出的IP地址（如 `10.0.2.18`），后续配置需要用到。

### 第2步：修改Docker配置中的代理地址
**重要提醒**：编辑 `ao3-proxy/docker-compose.yml` 文件，将 `HTTP_PROXY` 和 `HTTPS_PROXY` 的值中的IP地址替换为第1步获取的内网IP。

### 关于 main.ts 的关键配置
在 `ao3-proxy/main.ts` 文件中，确保 `PROXY_HOST` 变量正确设置为你的域名：
```typescript
const PROXY_HOST = Deno.env.get("PROXY_HOST") || "www.ao3.icu";
```
此配置影响Cookie重写和域名替换逻辑。

### 第3步：启动服务
```bash
# 启动Mihomo代理池
cd /root/proxy-pool
./start.sh

# 启动Docker反向代理
cd /www/docker/ao3-proxy
docker-compose up -d
```

### 第5步：测试访问
```bash
# 测试本地访问
curl -v http://localhost:8000/

# 测试代理功能
curl -x http://localhost:7890 https://api.ipify.org

# 查看服务状态
cd /root/proxy-pool && ./status.sh
cd /www/docker/ao3-proxy && docker-compose ps
```

## ⚠️ 常见问题排查

### 问题1：端口未监听
**症状**：`curl http://localhost:8000` 无响应

**检查命令：**
```bash
# 检查端口监听状态
ss -tulpn | grep -E '(7890|8000|9090)'

# 检查服务进程
ps aux | grep mihomo
docker-compose ps
```

**解决方案：**
1. 确保Mihomo已启动：`cd /root/proxy-pool && ./start.sh`
2. 确保Docker容器运行：`cd /www/docker/ao3-proxy && docker-compose up -d`
3. 检查配置文件中的端口是否正确

### 问题2：防火墙阻止
**症状**：外部无法访问，但本地可访问

**检查命令：**
```bash
# 查看防火墙状态
ufw status 2>/dev/null || systemctl status firewalld 2>/dev/null

# 临时测试（关闭防火墙）
ufw disable 2>/dev/null || systemctl stop firewalld 2>/dev/null
```

**解决方案：**
```bash
# 开放必要端口
ufw allow 8000/tcp  # 反向代理端口
ufw allow 7890/tcp  # 代理池端口
ufw allow 9090/tcp  # 管理API端口
```

## 🔄 日常维护命令

### IP切换管理
```bash
# 手动切换IP
cd /root/proxy-pool && ./switch.sh

# 查看当前IP
curl -x http://localhost:7890 https://api.ipify.org

# 查看切换日志
tail -f /root/proxy-pool/logs/switch_*.log
```

### 服务管理
```bash
# Mihomo服务
cd /root/proxy-pool
./start.sh    # 启动
./stop.sh     # 停止
./status.sh   # 状态

# Docker服务
cd /www/docker/ao3-proxy
docker-compose up -d    # 启动
docker-compose down     # 停止
docker-compose logs -f  # 查看日志
```

## 📝 部署检查清单
- [ ] Mihomo配置文件中的 `allow-lan: true` 已设置
- [ ] Mihomo配置文件中的 `external-controller: 0.0.0.0:9090` 已设置
- [ ] Docker配置中的 `HTTP_PROXY` 已更新为正确内网IP
- [ ] 服务器防火墙已开放8000、7890、9090端口
- [ ] 域名DNS解析已指向服务器IP
- [ ] 代理订阅链接仍然有效

## 💡 高级提示
1. **定时任务**：Mihomo已配置每10分钟自动切换IP，可通过crontab管理
2. **性能监控**：使用 `docker stats` 监控容器资源使用情况
3. **日志分析**：定期检查日志文件，分析访问模式和错误信息
4. **备份策略**：定期备份配置文件，特别是订阅链接和Cookie

## 🆘 紧急恢复
如果服务完全不可用：
1. 检查所有服务是否运行：`./status.sh` 和 `docker-compose ps`
2. 查看错误日志：`tail -f mihomo.log` 和 `docker-compose logs -f`
3. 重启所有服务：先 `./stop.sh` 再 `./start.sh`，然后 `docker-compose restart`
4. 检查网络连接：`ping` 和 `curl` 测试基本网络连通性

---

**最后提醒**：部署完成后，请务必测试完整的访问流程，包括页面浏览、登录功能和文件下载，确保所有功能正常工作。系统依赖外部代理订阅，请关注订阅有效期并及时更新。
