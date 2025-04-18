# SillyTavern 部署到 Google Cloud Run 指南 (使用 GCS 持久化数据)

本指南将引导您将 SillyTavern 应用程序部署到 Google Cloud Run。Cloud Run 是一个托管平台，可以运行您的容器化应用程序，并且可以自动扩展。我们将结合 Google Cloud Storage (GCS) 来存储 SillyTavern 的数据（如角色卡、聊天记录等），这样即使 Cloud Run 实例重启，您的数据也不会丢失。

**目标读者:** 本指南旨在让非技术用户也能理解并执行部署操作。

**核心原理:** 我们将 SillyTavern 打包成一个 Docker 镜像，上传到 Google Cloud，然后在 Cloud Run 上运行这个镜像。同时，我们会创建一个 GCS 存储桶（类似网络硬盘），并将其连接到 Cloud Run 服务，SillyTavern 配置为将所有数据读写到这个存储桶中。

---

## 准备工作 (Prerequisites)

在开始之前，请确保您已完成以下准备：

1.  **Google Cloud 账户:** 您需要一个 Google Cloud 账户。如果您没有，可以访问 [cloud.google.com](https://cloud.google.com/) 注册，新用户通常有免费试用额度。
2.  **启用结算:** Google Cloud Run 和 GCS 虽然有免费用量，但仍需要您在项目中启用结算。您可以在 Google Cloud Console 的"结算"部分完成此操作。不用担心，在免费额度内不会产生费用。
3.  **创建 Google Cloud 项目:** 在 Google Cloud Console 中创建一个新项目，或者选择一个现有项目。记下您的 **项目 ID (Project ID)**，后面会用到。
4.  **安装 `gcloud` 命令行工具:** 这是 Google Cloud 的官方命令行工具。访问 [Google Cloud SDK 文档](https://cloud.google.com/sdk/docs/install) 按照说明进行安装。安装后，打开您的终端（在 Windows 上是 Command Prompt 或 PowerShell，在 macOS 或 Linux 上是 Terminal），运行以下命令登录并设置您的项目：
    ```bash
    gcloud auth login
    gcloud config set project YOUR_PROJECT_ID
    # 将 YOUR_PROJECT_ID 替换为您真实的 Google Cloud 项目 ID
    ```
5.  **安装 Docker Desktop:** Docker 是用于构建和运行容器的工具。访问 [Docker 官网](https://www.docker.com/products/docker-desktop/) 下载并安装适合您操作系统的 Docker Desktop。安装后请启动 Docker Desktop。
6.  **准备 GCS 存储桶名称:** 想一个全局唯一的名称用于您的 GCS 存储桶（可以包含小写字母、数字、短划线）。例如 `sillytavern-data-yourname`。记下这个名称。

---

## 部署步骤

请按照以下步骤操作：

**第 1 步：构建 Docker 镜像**

Docker 镜像就像一个包含了 SillyTavern 应用程序及其运行环境的包裹。

1.  打开您的终端 (Terminal / Command Prompt)。
2.  使用 `cd` 命令进入到 SillyTavern 项目的根目录（包含 `Dockerfile` 文件的那个目录）。
3.  运行以下命令来构建镜像：
    ```bash
    docker build -t sillytavern .
    ```
    *   `-t sillytavern` 给镜像起个名字叫 `sillytavern`。
    *   `.` 表示使用当前目录下的 `Dockerfile` 来构建。
    *   这个过程可能需要几分钟时间，具体取决于您的网络和电脑性能。

**第 2 步：设置 Google Cloud Artifact Registry**

Artifact Registry 是 Google Cloud 上存放 Docker 镜像的地方。

1.  选择一个 Google Cloud 区域 (Region) 来存放您的镜像。例如 `us-central1` 或 `asia-east1`。记下您选择的区域。
2.  在终端运行以下命令创建 Artifact Registry 仓库 (如果还没有的话)：
    ```bash
    gcloud artifacts repositories create sillytavern-repo --repository-format=docker --location=YOUR_REGION --description="SillyTavern Docker repository"
    # 将 YOUR_REGION 替换为您选择的区域，例如 us-central1
    ```
3.  运行以下命令，让 Docker 可以登录到 Artifact Registry：
    ```bash
    gcloud auth configure-docker YOUR_REGION-docker.pkg.dev
    # 将 YOUR_REGION 替换为您选择的区域
    ```

**第 3 步：标记并推送 Docker 镜像**

现在，我们将本地构建好的镜像上传到 Artifact Registry。

1.  运行以下命令给镜像打上正确的标签：
    ```bash
    docker tag sillytavern YOUR_REGION-docker.pkg.dev/YOUR_PROJECT_ID/sillytavern-repo/sillytavern:latest
    # 将 YOUR_REGION 替换为您选择的区域
    # 将 YOUR_PROJECT_ID 替换为您真实的 Google Cloud 项目 ID
    ```
2.  运行以下命令将镜像推送到 Artifact Registry：
    ```bash
    docker push YOUR_REGION-docker.pkg.dev/YOUR_PROJECT_ID/sillytavern-repo/sillytavern:latest
    # 将 YOUR_REGION 和 YOUR_PROJECT_ID 替换为实际值
    ```
    这个过程可能也需要一些时间，具体取决于您的网络速度。

**第 4 步：创建 Google Cloud Storage (GCS) 存储桶**

这个存储桶将用来存放 SillyTavern 的数据。

1.  您可以通过 Google Cloud Console 界面操作：
    *   访问 Google Cloud Console ([console.cloud.google.com](https://console.cloud.google.com/))。
    *   在左侧导航菜单中找到 "Cloud Storage" -> "存储桶 (Buckets)"。
    *   点击 "创建 (Create)"。
    *   输入您之前准备好的 **唯一存储桶名称**。
    *   选择存储桶的 **位置 (Location)**，建议选择与您的 Cloud Run 服务相同的 **区域 (Region)**。
    *   其他选项可以使用默认设置，然后点击 "创建"。
2.  或者，您也可以使用终端命令创建 (更快捷)：
    ```bash
    gsutil mb -p YOUR_PROJECT_ID -l YOUR_REGION gs://YOUR_UNIQUE_BUCKET_NAME/
    # 将 YOUR_PROJECT_ID 替换为您的项目 ID
    # 将 YOUR_REGION 替换为您选择的区域
    # 将 YOUR_UNIQUE_BUCKET_NAME 替换为您准备好的存储桶名称
    ```

**第 5 步：部署到 Cloud Run**

这是最后一步，将您的镜像和服务配置部署到 Cloud Run。

1.  在终端运行以下命令：
    ```bash
    gcloud run deploy sillytavern-service \
        --image=YOUR_REGION-docker.pkg.dev/YOUR_PROJECT_ID/sillytavern-repo/sillytavern:latest \
        --platform=managed \
        --region=YOUR_REGION \
        --port=8080 \
        --allow-unauthenticated \
        --add-volume=name=gcs-data-vol,type=cloud-storage,bucket=YOUR_UNIQUE_BUCKET_NAME \
        --add-volume-mount=volume=gcs-data-vol,mount-path=/gcs/data \
        --min-instances=1 # 可选：保持至少一个实例运行以减少冷启动，但会产生少量持续费用
    # ---------- 请将以下占位符替换为您的实际值 ----------
    # YOUR_REGION: 您选择的 Google Cloud 区域 (例如 us-central1)
    # YOUR_PROJECT_ID: 您的 Google Cloud 项目 ID
    # YOUR_UNIQUE_BUCKET_NAME: 您创建的 GCS 存储桶名称
    # ----------------------------------------------------
    ```
    *   `sillytavern-service`: 您给 Cloud Run 服务起的名字。
    *   `--image`: 指向您刚刚推送到 Artifact Registry 的镜像。
    *   `--platform=managed`: 使用完全托管的 Cloud Run。
    *   `--region`: 指定服务运行的区域 (应与 GCS 存储桶和 Artifact Registry 仓库区域一致或相近)。
    *   `--port=8080`: 告诉 Cloud Run 您的应用程序在容器内监听 8080 端口。
    *   `--allow-unauthenticated`: **重要!** 这允许任何人访问您的 SillyTavern 服务。如果您只想自己访问，可以考虑移除此项并设置身份验证（但这会更复杂）。
    *   `--add-volume`: 定义一个名为 `gcs-data-vol` 的卷，它连接到您的 GCS 存储桶。
    *   `--add-volume-mount`: 将上面定义的卷挂载到容器内的 `/gcs/data` 目录。因为我们之前修改了 `config/config.yaml` 让 `dataRoot` 指向 `/gcs/data`，SillyTavern 现在会将数据写入 GCS。
    *   `--min-instances=1`: 保持至少一个实例运行。这可以避免应用在无人访问一段时间后完全关闭，从而减少下次访问时的等待时间（冷启动）。但这会产生少量持续费用，即使没人访问。如果不需要，可以移除此行。

2.  **授予权限:** Cloud Run 需要权限才能读写您的 GCS 存储桶。
    *   部署命令可能会提示您是否要为默认服务账号授予所需角色。通常可以直接回答 "y"。
    *   如果部署后遇到权限错误，您需要手动操作：
        *   找到 Cloud Run 服务的服务账号邮箱地址。您可以在 Cloud Console 的 Cloud Run 服务详情页的"安全"或"详情"标签页找到它，通常格式是 `PROJECT_NUMBER-compute@developer.gserviceaccount.com`。
        *   访问 GCS 存储桶页面，选择您的存储桶，进入"权限"标签页。
        *   点击"授予访问权限"。
        *   在"新的主账号"中输入服务账号邮箱地址。
        *   选择角色："Cloud Storage" -> "存储对象管理员 (Storage Object Admin)"。
        *   点击"保存"。

**第 6 步：访问您的 SillyTavern**

部署成功后，`gcloud` 命令会输出一个 **服务网址 (Service URL)**，类似 `https://sillytavern-service-xxxxxxxxxx-uc.a.run.app`。

将这个网址复制到您的浏览器中，您应该就能访问部署在 Cloud Run 上的 SillyTavern 了！由于数据存储在 GCS 上，您的角色、聊天记录等都会被持久保存。

---

## 注意事项与故障排除

*   **费用:** Google Cloud Run 和 GCS 都有每月免费用量。轻度使用可能完全免费。但如果流量很大或数据量很多，或者您设置了 `--min-instances=1`，可能会产生费用。请关注 Google Cloud 的"结算"页面。
*   **权限错误:** 如果遇到 "Permission Denied" 或 403 错误，通常是 Cloud Run 服务账号没有访问 GCS 存储桶的权限。请仔细检查**第 5 步**中的权限授予部分。
*   **部署失败:** 仔细阅读部署命令的错误输出，它通常会给出失败的原因。可能是镜像名称错误、区域不匹配、权限问题等。
*   **更新应用:** 如果您更新了 SillyTavern 代码，需要重新执行 **第 1 步** (构建镜像)、**第 3 步** (标记和推送新镜像) 和 **第 5 步** (重新部署 Cloud Run 服务，它会自动拉取最新的镜像)。

---

恭喜！您已成功将 SillyTavern 部署到 Google Cloud Run 并使用 GCS 进行数据持久化。 