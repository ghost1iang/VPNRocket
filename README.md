# 跨国网络无感无缝切换（回国/出国）终极部署说明书

经过数轮的测试与底层网络排查，我们最终抛弃了传统 VPN 方案（Tailscale 全局代理带来的网卡冲突和高延迟），转而采用了**「底层节点 + 客户端智能应用层分流」**的最佳实践方案。

本说明书详细记录了这套方案的运作逻辑，以及如何实现完全对称的“双向无感翻墙”。

---

## 核心架构设计原理

这套方案抛弃了系统级的虚拟网卡接管，改用**应用层透明代理**（类似高速公路的 ETC 专属通道）。
* **服务端（NAS）**：运行一个极轻量级的 Docker 容器（如 `xray-bridge` 或 `v2fly`），只负责在某个特定端口（如 `10086`）接收加密数据包，并将其解密后发送到本地互联网。
* **网络层（路由器）**：利用 DDNS（动态域名）绑定家庭公网 IP，并在主路由器上设置**端口转发**，将外网端口 `10086` 直通 NAS。
* **客户端（手机/Mac）**：运行 Shadowrocket（小火箭）。小火箭接管设备流量，并根据我们编写的**分流规则文件（`.conf`）**，在千分之一秒内决定：这个网站是该“直连”还是“打包发给海外节点”。

---

## 场景一：在澳洲如何“无感回国” (Australia ➡️ China)

**目标**：身在墨尔本，日常上网、看 Netflix、打澳洲本地游戏完全不受影响；但一打开腾讯视频、Bilibili、爱奇艺，自动瞬间穿越回成都，免除海外版权限制和卡顿。

### 1. 服务端配置（成都家）
* **NAS 部署**：成都 NAS 上运行了基于 `v2fly/v2fly-core` 或 `shadowsocks` 的代理节点，监听在内网的 `3128` 或 `10086` 端口。
* **网络打通**：成都电信自带公网 IP。群晖配置了 DDNS（如 `ghostliang.i234.me`）。成都的主路由器将公网的 `10086` 端口映射到了成都 NAS 的内网 IP 上。

### 2. 客户端配置（澳洲 Mac/手机）
* 在小火箭中添加一个节点，地址填写 `ghostliang.i234.me`，端口 `10086`，密码为您设置的专属密码。
* 导入并启用为您定制的 `SmartRoute.conf` 配置文件。

### 3. 智能分流逻辑 (`SmartRoute.conf` 节选)
```ini
[Rule]
# 遇到国内视频/音频网站的域名及其图片CDN，强制打包发给成都节点
DOMAIN-SUFFIX,qq.com,Auto_Switch
DOMAIN-SUFFIX,qpic.cn,Auto_Switch
DOMAIN-SUFFIX,iqiyi.com,Auto_Switch
DOMAIN-SUFFIX,bilibili.com,Auto_Switch

# 其余所有未匹配的流量（澳洲本地、国际网络等），全部走本地网卡直连
FINAL,DIRECT
```
**体验**：设置完毕后，小火箭常年挂在后台。您无需任何手工切换操作，真正实现“无体感”回国。

---

## 场景二：回国后如何“无感连澳洲” (China ➡️ Australia)

**目标**：过年回到成都，日常聊微信、看抖音走国内高速网络；但一打开 Google、Netflix、澳洲银行 App 时，自动穿梭回墨尔本家中的网络。

**操作方案：完全对称的镜像部署！**

### 1. 服务端配置（澳洲家）
* **NAS 部署**：在澳洲的群晖（`BaradineNAS`, `192.168.86.39`）上，用完全相同的 Docker 命令，部署一个 `xray-bridge` 节点，监听本地某个端口（例如 `10086`）。
* **网络打通**：
  1. 确保澳洲家庭宽带有公网 IP。给澳洲 NAS 配置一个 DDNS（例如 `baradine.i234.me`）。
  2. **（关键点）** 由于澳洲家是**双层路由NAT**（ISP 光猫 + Google WiFi），需要做两层端口映射：
     - ISP 路由器将 `10086` 映射给 Google WiFi (`192.168.1.100`)。
     - Google WiFi 将 `10086` 映射给澳洲 NAS (`192.168.86.39`)。

### 2. 客户端配置（国内 Mac/手机）
* 在小火箭中新增一个节点，地址填写澳洲的 DDNS `baradine.i234.me`，端口 `10086`。
* 新建一份名为 `AusRoute.conf` 的配置文件，逻辑刚好与 `SmartRoute` **相反**。

### 3. 智能分流逻辑 (`AusRoute.conf` 范例)
```ini
[Rule]
# 遇到外网或澳洲本地域名，强制发给澳洲节点
DOMAIN-SUFFIX,google.com,Auto_Switch
DOMAIN-SUFFIX,netflix.com,Auto_Switch
DOMAIN-SUFFIX,commbank.com.au,Auto_Switch

# 微信、淘宝、抖音等其余所有国内流量，直接走成都本地宽带
FINAL,DIRECT
```
**体验**：同样一键开启小火箭，无需繁琐切换，海内外网络自由穿梭。

---

## 场景三：系统运维与 SSH 后台管理

在抛弃了 Mac 端的 Tailscale 后（为了避免与小火箭发生网卡冲突），如果您需要对成都 NAS 进行底层代码维护，您无法直接 SSH 登录 `100.84.24.19`。

**最优雅的安全解决方案：跳板机模式 (Jump Host)**
澳洲 NAS 的后台 Tailscale 是常驻运行的（Linux 层面不冲突）。您可以把澳洲 NAS 当作一块安全跳板。

在 Mac 终端中，只需执行以下一行“套娃”命令：
```bash
ssh -t ghostliang@192.168.86.39 "ssh admin@100.84.24.19"
```
*(注：如果成都 NAS 启用了 ghostliang 的管理员权限，将 admin 替换为 ghostliang 即可)*

这样您就能在完全不修改本地网络、不暴露 22 端口的情况下，极其安全地连入成都 NAS 的底层系统了。

---

## 场景四：配置说明向导页的更新与部署

我们在澳洲 NAS 上部署了专属的配置下发与教程页面（基于 Web Station），供手机端快速访问和复制配置链接。
该网页的前端源码保存在 Mac 本地的 `VPNTailscale/web/` 目录下（包含 `index.html`, `editor.html`, `config/`, `images/` 等）。

### 如何手动更新网页并部署到群晖 NAS？
当您在本地修改了网页源码或增删了图片后，请按照以下标准流程将其部署到群晖：

1. **本地打包**：进入本地的 `web` 目录，将所有内容打包为 `.tar.gz` 压缩包。
   ```bash
   cd /Users/Leon/Documents/VPNTailscale/web
   tar -czf VPNRocket_update.tar.gz .
   ```
2. **传输至群晖**：将该压缩包上传到澳洲群晖的 `/var/services/web/VPNRocket/` 目录下。
3. **解压与赋权**：在群晖 SSH 中执行解压，并赋予 Web Station 运行账户 (`http:http`) 的权限。
   ```bash
   sudo tar -xzf VPNRocket_update.tar.gz -C /var/services/web/VPNRocket/
   sudo chown -R http:http /var/services/web/VPNRocket
   ```
4. **⚠️ 务必清理压缩包**：由于网页目录对外开放，**网页上传解压结束后，请务必立即删除该 `.tar.gz` 压缩包**，以防止配置源码或未公开文件被意外下载！
   ```bash
   sudo rm /var/services/web/VPNRocket/VPNRocket_update.tar.gz
   ```

---

## 场景五：本地开发环境维护 (TmpScript 清理)

在开发和自动部署过程中，我们会在本地工程根目录下（`VPNTailscale/TmpScript/`）生成许多用于自动化 SSH 操作的临时 Expect 脚本。
这些脚本内可能包含明文密码或服务器敏感信息。

**维护建议：**
请**不定期清理** `TmpScript` 文件夹中的所有 `.exp` 文件，以确保本地电脑的凭据安全。
```bash
# 在终端中执行以下命令清空临时脚本
rm -rf /Users/Leon/Documents/VPNTailscale/TmpScript/*
```
