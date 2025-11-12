# 在 macOS 上部署并测试语义视频片段检索

本文档详细介绍如何在 macOS 上拉起 Immich 的开发容器、上传示例影片、运行新的「视频片段语义索引」流水线，并通过 `/api/search/video-segments` 接口验证自然语言检索结果。步骤覆盖环境准备、服务启动、作业排程、调试排错与清理收尾，帮助你完整验证改动。

## 前置条件

- macOS 13（Ventura）或更高版本，建议 16 GB 以上内存及不少于 40 GB 的可用磁盘空间（用于 Docker 镜像、数据库卷和上传的媒体）。
- 已安装 [Docker Desktop for Mac](https://www.docker.com/products/docker-desktop)，并在系统偏好设置中启用了虚拟化。
- 通过 `xcode-select --install` 安装好 Xcode 命令行工具，以提供 `make` 等基础工具。
- 准备一段几分钟以上的视频文件（例如电影片段或预告片），确保足够的长度以生成多个语义片段。

> 默认的开发环境会以 CPU 模式运行机器学习服务；首次下载模型需要数分钟时间，无需额外 GPU。

## 仓库初始化

1. 克隆项目并进入目录：

   ```bash
   git clone https://github.com/immich-app/immich.git
   cd immich
   ```

2. 复制示例环境变量文件，调整存储路径：

   ```bash
   cp docker/example.env docker/.env
   open docker/.env
   ```

   至少需要把 `UPLOAD_LOCATION` 与 `DB_DATA_LOCATION` 指向本机可写目录，如：

   ```bash
   mkdir -p ~/immich-dev/uploads ~/immich-dev/database
   ```

   若计划复用既有素材，可在启动前把文件放入 `UPLOAD_LOCATION` 指定的目录。

## 启动开发容器

1. 以预设的 `make` 目标启动全套服务（Web、Server、Microservices、Machine Learning、PostgreSQL、Redis）：

   ```bash
   make dev
   ```

   首次执行会构建镜像，可能耗时 10–15 分钟。保持该终端开启以持续查看日志。

2. 另开新终端运行数据库迁移，创建 `video_segment` 表与向量索引：

   ```bash
   docker compose -f docker/docker-compose.dev.yml exec immich-server pnpm run migrations:run
   ```

   输出中出现 `Migration 1749000000000-AddVideoSegmentTable succeeded` 即代表成功。

3. 可选：实时查看后端日志，了解作业进度或错误：

   ```bash
   docker compose -f docker/docker-compose.dev.yml logs -f immich-server
   ```

## 上传测试数据

1. 在浏览器访问 http://localhost:3000 ，注册首个账号（会自动成为管理员）。
2. 在 Web 页面通过「Upload」按钮上传准备好的视频，等待进度条结束。
3. 进入 **Administration → Jobs** 页面可监控后台队列的执行状态。

## 运行视频分段作业

1. 在 Web 后台的 **Administration → Jobs** 中找到 **Video Segmentation** 队列，点击 **Start** 启动作业，保持页面开启直至状态回到「Idle」。
   - 若偏好使用 API，可发送认证请求：

     ```bash
     curl -X PUT \
       -H 'Content-Type: application/json' \
       -H 'x-api-key: <你的管理员 API Key>' \
       -d '{"command":"start","force":false}' \
       http://localhost:2283/api/jobs/videoSegmentation
     ```

     管理员 API Key 可在 Web 端 **Profile → API Keys** 页面创建。

2. 观察终端日志。若某段视频解析失败，队列会给出警告；若一切正常，作业会静默处理所有待办资产。

3. 确认数据库中生成了片段记录：

   ```bash
   docker compose -f docker/docker-compose.dev.yml exec database \
     psql -U ${DB_USERNAME} ${DB_DATABASE_NAME} -c "SELECT COUNT(*) FROM video_segment;"
   ```

   将 `${DB_USERNAME}`、`${DB_DATABASE_NAME}` 替换为 `docker/.env` 中的配置。返回值大于 0 表示索引成功。

## 使用自然语言检索片段

1. 再次前往 **Profile → API Keys** 复制管理员 API Key。
2. 向新接口发出语义检索请求，`size` 控制返回条数：

   ```bash
   curl -X POST \
     -H 'Content-Type: application/json' \
     -H 'x-api-key: <你的管理员 API Key>' \
     -d '{"query":"枪战","size":10}' \
     http://localhost:2283/api/search/video-segments
   ```

3. 检视 JSON 响应：
   - `asset`：母视频的基础信息（ID、文件名、所有者等）。
   - `startTime` / `endTime`：片段的起止秒数。
   - `confidence`：基于余弦相似度的置信分数（0–1）。

4. 将时间戳导入剪辑软件或播放器，确认检索结果是否匹配关键词。

## 排错建议

- **机器学习服务不可用**：使用 `docker compose ... ps` 确认 `immich_machine_learning` 状态，并检查内存是否充足。重新启动容器通常可恢复模型下载。
- **返回空结果**：确认分段队列已完成，且视频长度足以切出多个 10 秒片段。可在 Jobs 页勾选 **Force** 再次运行，或把 `{"command":"start","force":true}` 发送至队列接口。
- **向量索引报错**：若迁移出现向量扩展错误，可删除数据库卷（`rm -rf $(grep DB_DATA_LOCATION docker/.env | cut -d= -f2)/*`），再执行 `make dev` 重新初始化数据库。

## 停止与清理

测试完成后，在执行 `make dev` 的终端按 `Ctrl+C` 终止容器；如需彻底清理，可运行：

```bash
docker compose -f docker/docker-compose.dev.yml down --remove-orphans
```

上传的媒体保存在 `docker/.env` 配置的目录，如需重置环境，请手动删除相应文件夹。
