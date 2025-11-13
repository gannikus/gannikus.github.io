---
title: "trueNas代理"
date: 2023-11-12T10:00:00+08:00
draft: false  # 确保这不是草稿
author: "Gannicus"
# --- 重点在这里 ---
categories:
    - "trueNas"
tags:
    - "mihomo"
    - "proxy"
    - "trueNas"
# --------------------
---本指南总结了在 TrueNAS SCALE Dragonfish (24.04+) 版本上安装并正确配置 `mihomo` (Clash.Meta 核心) 及 `yacd` (WebUI 仪表盘) 的**最终正确步骤**。

本指南的核心是**绕过** TrueNAS 服务器本身的网络限制（无法拉取 Docker 镜像），**解决**浏览器安全策略（CORS/PNA）导致的连接失败，并**修复**配置文件（`GeoSite.dat` / `fake-ip`）的启动崩溃问题。
## 目录

1. **准备工作**: 创建数据集并设置 SMB 共享
    
2. **步骤一: 解决 NAS 的“鸡生蛋”网络问题** (使用 PC 代理)
    
3. **步骤二: 准备三个关键配置文件** (`config.yaml`, `Country.mmdb`, `GeoSite.dat`)
    
4. **步骤三: 上传文件并修复权限** (关键！)
    
5. **步骤四: 安装 `mihomo` 引擎** (App 1)
    
6. **步骤五: 安装 `yacd` 仪表盘** (App 2)
    
7. **步骤六: 验证与使用** (连接仪表盘)
    
8. **步骤七: 让 NAS 自己走代理** (最终目标)

## 准备工作: 创建数据集并设置 SMB 共享

您需要一个地方来**永久存放**您的代理配置。我们必须使用 SMB 共享，因为您需要从 Windows 电脑向 NAS 传输文件。

1. **创建数据集 (Dataset):**
    
    - 前往 `Datasets` (数据集)。
        
    - 在您的主存储池下（例如 `MyPool`），创建一个专用于存放应用配置的数据集。
        
    - 路径示例: `/mnt/MyPool/AppsConfig`
        
    - 在它下面，再创建一个专门给 `mihomo` 用的数据集。
        
    - **最终路径示例: `/mnt/MyPool/AppsConfig/ClashConfig`**
        
2. **设置 SMB 共享:**
    
    - 前往 `Shares` (共享) -> `Windows (SMB)`。
        
    - 点击 `Add` (添加)。
        
    - `Path` (路径): 浏览并选择您刚刚创建的 `AppsConfig` 数据集 (`/mnt/MyPool/AppsConfig`)。
        
    - `Name` (名称): `AppsConfig` (或其他您喜欢的名字)。
        
    - `Purpose` (用途): `Default share parameters`。
        
    - `Save` (保存)。
        

## 步骤一: 解决 NAS 的“鸡生蛋”网络问题 (使用 PC 代理)

这是最关键的一步。您的 NAS 因为网络问题无法下载应用镜像。我们将让 NAS 通过您**有代理的电脑**来下载。

1. **在您的 Windows 电脑上:**
    
    - 确保您的代理软件 (Clash, V2RayN 等) **正在运行**。
        
    - 确保您电脑上的 **Tailscale 正在运行**。
        
    - 在您的代理软件设置中，**勾选 "Allow LAN" (允许局域网连接)**。
        
    - 找到您电脑的 **Tailscale IP 地址** (例如 `100.123.45.67`)。
        
    - 找到您代理软件的 **HTTP 代理端口** (例如 `7890`)。
        
2. **在您的 TrueNAS 网页界面:**
    
    - 前往 `System Settings` (系统设置)。
        
    - 点击 **`Apps` (应用)** (那张卡片)。
        
    - 找到 `Proxy` (代理) 设置。
        
    - **`HTTP Proxy`**: 填入 `http://[您PC的Tailscale-IP]:[您的代理端口]`
        
        - 示例: `http://100.123.45.67:7890`
            
    - **`HTTPS Proxy`**: 填入相同的内容。
        
        - 示例: `http://100.123.45.67:7890`
            
    - 点击 `Save` (保存)。
        

您的 NAS 现在会通过您电脑的代理来拉取所有应用镜像。

## 步骤二: 准备三个关键配置文件

您**不能**使用“最小化订阅”配置，因为它在启动时无法下载节点。您**必须**使用一个**完整**的静态配置文件。

请在您**有代理的 Windows 电脑**上准备以下三个文件：

1. **`config.yaml` (主配置文件):**
    
    - 在浏览器中（走代理）**直接访问**您的 Clash 订阅 URL。
        
    - 将显示的所有文本**复制**到一个新文件中，保存为 `config.yaml`。
        
    - **（关键！）** 打开这个 `config.yaml` 文件，**手动修改**以下内容：
        
        - **移除 `fake-ip`:** 找到 `dns:` 部分，删除 `enhanced-mode: fake-ip` 和 `fake-ip-range:` 这两行。这可以防止与 NAS 系统代理冲突。
            
        - **添加 `CORS`:** 找到 `external-controller: '0.0.0.0:9090'` 这一行，在它**下面**添加：
            
            YAML
            
            ```
            cors-allowed-origins:
              - 'http://192.168.4.195:8080' # 允许您 NAS 上的 YACD 访问
              - 'http://yacd.metacubex.one'  # 允许公共 YACD 访问 (备用)
            ```
            
            _(请将 `192.168.4.195` 替换为您 NAS 的真实 IP)_
            
2. **`Country.mmdb` (GEOIP 规则文件):**
    
    - 在浏览器中下载这个文件 (来自可靠的 `Loyalsoldier` 仓库):
        
    - [https://github.com/Loyalsoldier/v2dat/releases/latest/download/geoip.dat](https://www.google.com/search?q=https://github.com/Loyalsoldier/v2dat/releases/latest/download/geoip.dat)
        
    - **（注意！）** 下载后，将文件名 `geoip.dat` **重命名**为 `Country.mmdb`。`mihomo` 需要这个名字。
        
3. **`GeoSite.dat` (GEOSITE 规则文件):**
    
    - 在浏览器中下载这个文件 (同样来自 `Loyalsoldier`):
        
    - [https://github.com/Loyalsoldier/v2dat/releases/latest/download/geosite.dat](https://www.google.com/search?q=https://github.com/Loyalsoldier/v2dat/releases/latest/download/geosite.dat)
        

## 步骤三: 上传文件并修复权限 (关键！)

1. **上传文件:**
    
    - 在 Windows 电脑上，打开文件资源管理器。
        
    - 访问您的 SMB 共享 (例如 `\\192.168.4.195\AppsConfig`)。
        
    - 进入 `ClashConfig` 文件夹。
        
    - 将您准备好的**三个文件** (`config.yaml`, `Country.mmdb`, `GeoSite.dat`) **全部**复制粘贴到这个文件夹中。
        
2. **修复权限:**
    
    - 回到 TrueNAS **Shell** (命令行)。
        
    - **`cd`** 到您的配置目录 (请使用您的真实路径)：
        
        Bash
        
        ```
        cd /mnt/MyPool/AppsConfig/ClashConfig
        ```
        
    - 运行**权限修复**命令，将这三个文件的所有权交给 `apps` 用户 (UID 568)：
        
        Bash
        
        ```
        chown -R 568:568 .
        ```
        
        _(注意：命令末尾有一个 `.`，代表“当前目录”)_
        

## 步骤四: 安装 `mihomo` 引擎 (App 1)

1. 回到 TrueNAS `Apps` -> `Discover Apps` (发现应用)。
    
2. 点击 `⋮` (三点菜单) -> `Install via YAML`。
    
3. 给应用起个名字 (例如 `mihomo`)。
    
4. **粘贴以下 YAML**:
    
    YAML
    
    ```
    services:
      mihomo:
        container_name: mihomo
    
        # 我们从 GitHub 仓库拉取
        image: ghcr.io/metacubex/mihomo:latest
    
        # 使用 Host 网络模式
        network_mode: "host"
        privileged: true
    
        # 挂载您的配置目录
        volumes:
          # ！！！请将 : 左边的路径修改为您真实的路径！！！
          - /mnt/MyPool/AppsConfig/ClashConfig:/data
    
        # 告诉 mihomo 配置文件在 /data
        command: "-d /data"
    
        restart: unless-stopped
    ```
    
5. 点击 `Install`。由于您在【步骤一】中配置了 PC 代理，这次拉取将会成功。
    

## 步骤五: 安装 `yacd` 仪表盘 (App 2)

为了**绕过**浏览器的 PNA/CORS 安全限制，我们必须在 NAS 本地安装 WebUI。

1. 再次点击 `Install via YAML`。
    
2. 给应用起个名字 (例如 `yacd-dashboard`)。
    
3. **粘贴以下 YAML**:
    
    YAML
    
    ```
    services:
      yacd:
        container_name: yacd-dashboard
    
        # YACD 仪表盘镜像
        image: haishanh/yacd:latest
    
        # 将 NAS 的 8080 端口映射到容器的 80 端口
        ports:
          - "8080:80"
    
        restart: unless-stopped
    ```
    
4. 点击 `Install`。
    

## 步骤六: 验证与使用 (连接仪表盘)

您现在**万事俱备**。

1. 打开您的浏览器。
    
2. **访问您本地的 YACD 仪表盘**：
    
    > `http://[您的NAS-IP]:8080` (例如: `http://192.168.4.195:8080`)
    
3. 您会看到 YACD 界面。在右上角或弹窗中，输入 `mihomo` 引擎的 API 地址：
    
    > `http://[您的NAS-IP]:9090` (例如: `http://192.168.4.195:9090`)
    
4. 点击“连接”。
    

**您现在应该能看到 YACD 成功连接，并显示出您 `config.yaml` 中的所有节点！**

## 步骤七: 让 NAS 走代理 (最终目标)

现在 `mihomo` 已经在 `127.0.0.1:7890` (本机) 上提供了代理，您可以配置 NAS 的各个部分来使用它。

1. **让 TrueNAS 系统后台走代理:**
    
    - 前往 `System Settings` (系统设置) -> `General` (常规)。
        
    - `HTTP Proxy`: `http://127.0.0.1:7890`
        
    - `HTTPS Proxy`: `http://127.0.0.1:7890`
        
    - `Save` (保存)。
        
    - _(注意：这不会影响您的 Shell，Shell 测试必须用 `curl --proxy http://127.0.0.1:7890 https://google.com`)_
        
2. **让 qBittorrent 走代理:**
    
    - 打开 qBittorrent WebUI。
        
    - `工具` -> `选项` -> `连接` -> `代理服务器`。
        
    - `类型`: **SOCKS5**
        
    - `主机`: `192.168.4.195` (qBit 必须用 IP 访问 NAS 上的 `mihomo`)
        
    - `端口`: `7890`
        
3. **让您的 Tailscale 手机走代理:**
    
    - 在您的手机上，打开 Tailscale 应用。
        
    - `Preferences` -> `HTTP proxy`。
        
    - 填入: `http://[您的NAS的Tailscale-IP]:7890`
        
    - _(NAS 的 Tailscale IP 可以通过在 NAS Shell 运行 `tailscale ip -4` 查到)_