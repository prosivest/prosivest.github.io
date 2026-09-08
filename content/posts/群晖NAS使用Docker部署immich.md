---
date: 2026-09-08T15:55:09+08:00
lastmod: 2026-09-08T15:55:09+08:00
title: "群晖NAS使用Docker部署immich"
description: "记录在群晖 NAS 上使用 Docker 部署 immich 的完整步骤，包括环境变量配置、容器启动和照片存储路径设置，并结合反向地理编码汉化方案提升中文体验。"
author: ["想上天的鱼"]
tags: ["immich", "docker", "NAS", "群晖", "照片管理", "反向地理编码"]
ShowToc: true
draft: false
---

# 部署

Immich 作为一款开源的高性能自托管照片和视频备份解决方案，提供了类似 Google Photos 的使用体验，以在自己的 NAS 上安全地管理所有照片。本文记录了我根据 Immich 官方指南在群晖 NAS 上使用 Docker 部署 Immich，并结合 ZingLix/immich-geodata-cn 项目实现反向地理编码汉化的全过程。

## 一、Immich 简介

Immich 是一款自托管的开源照片管理工具，支持从手机浏览器或专用 App（Android/iOS）上传照片和视频。它最大的优势在于：

- **数据自主可控**：所有照片存储在自己的 NAS 上，无需担心云服务商的隐私政策
- **功能丰富**：支持人脸识别、智能搜索、地图模式等
- **高性能**：专为自托管场景优化

不过，Immich 默认的反向地理编码功能输出的是英文地点名称，对于国内用户来说不太友好。这也是我后续进行汉化的原因。

## 二、在群晖NAS上部署Immich

### 2.1 准备工作

首先，确保你的群晖 NAS 已经安装了 **Container Manager**（在套件中心中安装）。

### 2.2 创建目录结构

在 File Station 中，进入 `docker` 文件夹，创建一个名为 `immich` 的目录。然后在其中创建以下子目录：

```
./docker/immich/
├── postgres/
└── library/
```

其实library目录没有用，因为后续将`UPLOAD_LOCATION`照片存储路径改为了`/volume2/Photos` 。

### 2.3 下载配置文件

从 Immich 官方 GitHub 下载最新版本的 `docker-compose.yml` 和 `example.env` 两个文件，上传到 `./docker/immich/` 目录中，并将 `example.env` 重命名为 `.env`。

### 2.4 配置 .env 文件

编辑 `.env` 文件，自定义以下关键配置：

- `DB_PASSWORD`：数据库密码
- `DB_USERNAME`：数据库用户名
- `UPLOAD_LOCATION`：照片存储路径
- `TZ`：时区设置

- 具体文件内容为：

  ```bash
  # You can find documentation for all the supported env variables at https://docs.immich.app/install/environment-variables

  # The location where your uploaded files are stored
  UPLOAD_LOCATION=/volume2/Photos

  # The location where your database files are stored. Network shares are not supported for the database
  DB_DATA_LOCATION=./postgres

  # To set a timezone, uncomment the next line and change Etc/UTC to a TZ identifier from this list: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones#List
  TZ=Asia/Shanghai

  # The Immich version to use. You can pin this to a specific version like "v2.1.0"
  IMMICH_VERSION=v3

  # Connection secret for postgres. You should change it to a random password
  # Please use only the characters `A-Za-z0-9`, without special characters or spaces
  DB_PASSWORD=immich

  # The values below this line do not need to be changed
  ###################################################################################
  DB_USERNAME=postgres
  DB_DATABASE_NAME=immich
  ```

着重两个地方进行了修改：

1. `UPLOAD_LOCATION=/volume2/Photos` 上传的路径设置在了Photos文件夹下
2. `TZ=Asia/Shanghai` 时区设置为东八区

### 2.5 配置 docker-compose.yml 文件

`docker-compose.yml` 文件中有以下几处修改：

1. 在immich-server中volumes下增加以下内容

```bash
volumes:
      # Do not edit the next line. If you want to change the media storage location on your system, edit the value of UPLOAD_LOCATION in the .env file
      - ${UPLOAD_LOCATION}:/data
      - /etc/localtime:/etc/localtime:ro
      - /volume2/Old photos/homes:/volume2/Old photos/homes:ro
      - /volume2/docker/immich/geodata:/build/geodata
      - /volume2/docker/immich/i18n-iso-countries/langs:/usr/src/app/server/node_modules/i18n-iso-countries/langs
```

主要是两部分：添加外部库和反向地理编码汉化。添加外部库参考`- /volume2/Old photos/homes:/volume2/Old photos/homes:ro` 按`宿主机路径：容器内路径：权限` 配置后，在immich设置中添加，3.1 外部库 。地理编码汉化见**四、反向地理编码汉化** 。

1. 数据库不是存储在 SSD 上，取消注释 `DB_STORAGE_TYPE: 'HDD'` 这一行。

### 2.6 在 Container Manager 中创建项目

1. 打开 Container Manager，点击左侧导航栏的 **Project**，然后点击 **Create**
2. 设置 **Project name** 为 `immich`
3. 在 **Path** 中选择之前创建的 `./docker/immich/` 目录
4. 系统会自动识别已存在的 `docker-compose.yml`，点击 **OK** 继续

完成后，Container Manager 会自动拉取镜像并启动所有容器。

### 2.7 配置防火墙

项目构建完成后，需要配置群晖的防火墙规则，允许 Immich 容器对外通信：

1. 打开 **控制面板 → 安全 → 防火墙**
2. 点击 **编辑规则**，添加以下规则：
   - **Source IP**：填入 Immich 容器的 IP 地址（在 Container Manager 中查看容器详情获取）
   - **Ports**：添加端口 `2283`（TCP）

配置完成后，通过 `http://你的群晖IP:2283` 即可访问 Immich。

## 三、immich配置

### 3.1 外部库

在系统设置-外部库中添加即可，文件夹填写对应目录即可。

### 3.2 手机app

安装immich手机app，设置同步的相册，同步即可。https://docs.immich.app/features/mobile-backup

## 四、反向地理编码汉化

Immich 默认识别出的照片位置信息是英文的，国内地址还有不少奇奇怪怪的非标准地名。ZingLix 开发的 immich-geodata-cn 项目完美解决了这个问题。

### 4.1 项目特点

这个项目的主要功能包括：

- **地点中文化**：对国内地址（含港澳台）实现完整汉化，实验性支持部分海外地区（目前包括日本）
- **地址标准化**：通过高德/Nominatim API，将地址统一规范为一级、二级、三级、四级行政区四个层级
- **自动更新**：仓库每周定期拉取最新地理信息数据并自动发布更新版本

### 4.2 下载数据文件

在项目的 Release 页面 下载 `geodata.zip` 和 `i18n-iso-countries.zip` 两个文件并解压。

数据版本说明：

- **自动更新版**：更新更频繁，推荐使用
- **手动发布版**：相对稳定，适合作为备用
- 文件名以 `_full` 结尾的版本数据量更大，在边界位置识别上表现更好，但识别速度可能会稍慢
- 最终下载的为`geodata_admin_2_admin_3_admin_4.zip`和`i18n-iso-countries.zip`

将解压后的 `geodata` 文件夹和 `i18n-iso-countries` 文件夹放到 `./docker/immich/` 目录下。

### 4.3 配置 Docker 容器映射

修改 `docker-compose.yml` 文件，在 `immich-server` 服务的 `volumes` 中添加以下挂载：

```yaml
volumes:
  - /volume2/docker/immich/geodata:/build/geodata
  - /volume2/docker/immich/i18n-iso-countries/langs:/usr/src/app/server/node_modules/i18n-iso-countries/langs
```

> **注意**：以上配置适用于 Immich 版本 >= 1.136.0。如果使用的是更早的版本，请参考项目 README 中的说明。

### 4.4 重启 Immich

在 Container Manager 中重新启动 Immich 项目，或通过命令行执行：

```bash
docker compose down && docker compose up -d
```

重启后检查启动日志，如果看到类似 `10000 geodata records imported` 的日志，说明 geodata 已成功加载。如果没有出现该日志，可以尝试修改 `geodata/geodata-date.txt` 文件中的日期为当前时间，Immich 只会在文件日期新于上次加载时更新数据。

### 4.5 刷新照片元数据

登录 Immich 管理后台，进入 **系统管理 → 任务** 页面，点击 **提取元数据** 中的 **全部**，触发对所有照片位置信息的刷新。

等待任务完成后，所有照片的位置信息就会显示为中文地名，并且支持用中文地名进行搜索。后续新增的照片无需任何额外操作，会自动应用中文地理编码。

## 五、immich调用核显

### 5.1 启用机器学习（ML）的硬件加速

群晖NAS使用的CPU是intel G5400T 是第 8 代（Coffee Lake）CPU，核显是UHD Graphics 610，应该使用 **OpenVINO** 后端。

1. **下载官方扩展文件**

需要将官方提供的`hwaccel.ml.yml` 文件下载到 Immich 项目目录（即`docker-compose.yml`所在文件夹）。

1. **修改 `docker-compose.yml`**

打开 `docker-compose.yml` 文件，在 `immich-machine-learning` 服务下进行两处修改：

- **修改镜像标签**：在 `image` 字段的末尾，将默认的 `cpu` 改为 `openvino`。
- **启用扩展配置**：取消 `extends` 部分的注释，并将 `service` 的值从 `cpu` 改为 `openvino`。

```yaml
immich-machine-learning:
  container_name: immich_machine_learning
  # For hardware acceleration, add one of -[armnn, cuda, rocm, openvino, rknn] to the image tag.
  # Example tag: ${IMMICH_VERSION:-release}-cuda
  image: ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION:-release}-openvino
  extends: # uncomment this section for hardware acceleration - see https://docs.immich.app/features/ml-hardware-acceleration
    file: hwaccel.ml.yml
    service: openvino # set to one of [armnn, cuda, rocm, openvino, openvino-wsl, rknn] for accelerated inference - use the `-wsl` version for WSL2 where applicable
```

1. **重新部署容器**

修改并保存文件后，通过以下命令重新创建并启动容器，使配置生效：

```
docker compose up -d
```

1. **验证是否生效**

可以通过以下两种方式确认硬件加速是否正常工作：

- **检查设备访问**：运行以下命令，确认容器可以访问核显设备 `/dev/dri`：
  ```
  docker exec -t immich_machine_learning ls -la /dev/dri
  ```
  如果能看到 `card0` 和 `renderD128` 等文件，说明设备挂载成功。
- **检查容器日志**：查看 `immich-machine-learning` 的日志。当执行人脸识别或智能搜索任务时，日志中应该会显示 `Available ORT providers` 并包含 `OpenVINOExecutionProvider` 相关信息。

### 5.2 启用硬件转码

- **CPU 要求**： G5400T 是第 8 代（Coffee Lake）CPU，文档指出 **VP9 编码需要第 9 代或更新的 CPU**。对于 H.264 和 HEVC 编码，通常是可以支持的。
- **系统要求**：文档明确说明 **WSL2 不支持 Quick Sync**。你使用的是群晖 NAS 的 Docker 部署，这属于 Linux 环境，是支持的。
- **核显设备**：需要确认群晖 NAS 的内核已加载核显驱动，并在 `/dev/dri` 路径下生成了 `renderD128` 等设备文件。

配置的核心思路是：**下载官方扩展文件 → 修改 `docker-compose.yml` → 在网页管理界面启用**。

**1. 下载官方扩展文件**

将官方提供的 `hwaccel.transcoding.yml` 文件下载到 Immich 项目目录（即 `docker-compose.yml` 所在文件夹）

**2. 修改 `docker-compose.yml`**

打开 `docker-compose.yml` 文件，在 `immich-server` 服务下进行修改：

- **取消注释 `extends` 部分**。
- 将 `service` 的值从 `cpu` 改为 `quicksync`（代表 Intel Quick Sync）。

修改后的 `immich-server` 服务配置如下：

```yaml
immich-server:
  container_name: immich_server
  image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}
  extends:
    file: hwaccel.transcoding.yml
    service: quicksync # set to one of [nvenc, quicksync, rkmpp, vaapi, vaapi-wsl] for accelerated transcoding
  # ... 其他配置 ...
```

**3. 重新部署容器**

修改并保存文件后，通过以下命令重新创建并启动容器，使配置生效：

```bash
docker compose up -d
```

**4. 在 Immich 网页界面启用**

- 以管理员身份登录 Immich 网页。
- 进入 **管理员设置（Administration）** -> **视频转码设置（Video Transcoding Settings）**。
- 在 **硬件加速（Hardware Acceleration）** 下拉菜单中选择 **`Quick Sync`**。
- （可选）在 **硬件解码（Hardware decoding）** 选项中启用，以获得端到端的硬件加速。
- 保存设置。

**5. 验证是否生效**

- **检查设备访问**：运行以下命令，确认容器可以访问核显设备：
  如果能看到 `card0` 和 `renderD128` 等文件，说明设备挂载成功。
  `bash
docker exec -t immich_server ls -la /dev/dri
`
- **监控 GPU 使用率**：在触发转码任务（如上传新视频）时，在群晖 NAS 上使用 `intel_gpu_top` 等工具查看核显使用率。如果看到 GPU 有活动，说明加速生效。
- **检查容器日志**：查看 `immich-server` 的日志，在转码时不应出现与硬件加速相关的错误信息。

## 六、后续维护

### 6.1 更新地理数据

项目数据会定期更新，可以手动下载最新 Release 文件替换后重启 Immich。

也可以使用项目提供的自动更新脚本：

```bash
# 进入 geodata 目录
cd /path/to/immich-app/geodata

# 下载脚本
curl -o update.sh <https://raw.githubusercontent.com/ZingLix/immich-geodata-cn/refs/heads/main/geodata/update.sh>

# 运行更新（参数可选 geodata_admin_2、geodata_admin_3 等）
bash update.sh geodata_admin_2
```

更新完成后需要重启 Immich 重新加载数据。也可以设置定时任务自动更新：

```bash
5 5 * * 6 bash /immich_data/geodata/update.sh && docker restart immich_server
```

### 6.2 升级 Immich

当 Immich 发布新版本时，只需在 Container Manager 中重新拉取镜像并重建项目即可。

## 七、后续需解决问题

暂无

---

**参考链接**：

- Immich 官方文档：https://immich.app/docs/install/docker-compose
- Immich 群晖安装指南：https://docs.immich.app/install/synology/
- immich-geodata-cn 项目：https://github.com/ZingLix/immich-geodata-cn
