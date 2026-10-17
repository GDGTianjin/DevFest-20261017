# Google Cloud Run + Neon + Gmail SMTP 部署 n8n 实战指南（含表单自动发信工作流）

本指南提供一套**零固定月费成本、免运维、生产可用**的 n8n 自动化系统搭建方案。通过结合 **Google Cloud Run**（无服务器容器）、**Neon**（Serverless PostgreSQL 数据库）与 **Gmail SMTP**，你可以快速上线一个“用户填写网页表单 -> 自动发送预设 HTML 邮件”的完整工作流。

---

## 架构常见坑点

在 Cloud Run 上运行 n8n 常见的三个大坑：
1. **容器重启数据丢失**：弃用本地 SQLite，直连外部免费的 **Neon PostgreSQL 连接池（Pooler）**，实现工作流与凭证 100% 持久化。
2. **`/healthz` 路由冲突**：Cloud Run 内部保留了 `/healthz` 路径，指南中通过设置 `N8N_ENDPOINT_HEALTH=health` 避免画布连接异常。
3. **25 端口封禁与 OAuth 过期**：GCP 默认封禁出站 25 端口，且 Gmail OAuth 测试模式每 7 天强制失效；本方案采用 **Gmail SMTP (465 SSL 端口) + 应用专用密码**，配置一次永久有效。

---

## 第一步：创建 Neon Serverless PostgreSQL 数据库

1. 访问并登录 [Neon 控制台](https://console.neon.tech/)。
2. 点击 **New Project** 创建一个项目：
   * **Project Name**: `n8n-db`
   * **Database Name**: `neondb`（默认即可）
   * **Region**: 选择离你后面 Cloud Run 节点最近的区域（例如 `AWS Asia Pacific (Singapore) - ap-southeast-1`）。
3. 创建成功后，在项目的 **Dashboard -> Connection Details** 面板中：
   * **关键操作**：确保开启/勾选 **Pooled connection**（连接池模式，主机名会带有 `-pooler` 后缀，防止 Serverless 容器扩缩容耗尽连接数）。
   * 将展示的的连接串拆解并记录以下 4 个参数备用：
     * **NEON_HOST**: 形如 `ep-cool-darkness-123456-pooler.ap-southeast-1.aws.neon.tech`
     * **NEON_DB**: `neondb`
     * **NEON_USER**: 形如 `neondb_owner`
     * **NEON_PASS**: 你的数据库密码

---

## 第二步：获取 Gmail 16 位应用专用密码 (App Password)

为了绕过复杂的 OAuth 审批并实现永久稳定发信，我们使用 Google 官方的“应用专用密码”：

1. 确保你的 Google 账号已经开启了 **两步验证 (2-Step Verification)**。
2. 直接访问 [Google 应用专用密码管理页](https://myaccount.google.com/apppasswords)。
3. 在 **App name (应用名称)** 输入框中填写 `n8n-cloud-run`，点击 **Create (创建)**。
4. 弹窗中会显示一串 **16 位字母密码**（格式如 `abcd efgh ijkl mnop`）。
5. 复制这 16 位字母并**去掉中间的所有空格**（变成 `abcdefghijklmnop`），妥善保存备用。

---

## 第三步：一键部署 n8n 到 Google Cloud Run

1. 打开 [Google Cloud Console](https://console.cloud.google.com/)，确保你的项目已绑定结算账户（享有每月免费额度）。
2. 点击页面右上角的图标激活 **Cloud Shell**（或者在本地已登录 `gcloud` 的终端中执行）。
3. 创建部署脚本文件 `deploy-n8n.sh`：

也可以直接访问: https://gdg20261017.n8nbook.com/ 来获取到生成器

```bash
cat << 'EOF' > deploy-n8n.sh
#!/bin/bash
set -e

# ================= 1. 请修改以下 7 个配置变量 =================
export PROJECT_ID="你的-gcp-项目-id"
export REGION="asia-east1"                  # 台湾节点（或选新加坡 asia-southeast1）
export SERVICE_NAME="n8n"

# 填入第一步获取的 Neon 数据库连接信息 (务必带 -pooler)
export NEON_HOST="ep-xxx-pooler.ap-southeast-1.aws.neon.tech"
export NEON_DB="neondb"
export NEON_USER="neondb_owner"
export NEON_PASS="你的Neon数据库密码"

# 填入第二步获取的 Gmail 信息
export GMAIL_USER="你的邮箱@gmail.com"
export GMAIL_APP_PASS="你的16位应用专用密码(无空格)"
# ===============================================================

echo "🚀 [1/4] 设置 GCP 项目并启用必要 API..."
gcloud config set project $PROJECT_ID
gcloud services enable run.googleapis.com secretmanager.googleapis.com

echo "🔐 [2/4] 创建并配置 Secret Manager 敏感密钥..."
# 生成固定的 n8n 凭证加密密钥（若已存在则跳过，切勿轻易覆盖）
if ! gcloud secrets describe n8n-encryption-key >/dev/null 2>&1; then
  openssl rand -base64 32 | gcloud secrets create n8n-encryption-key --data-file=-
fi

# 写入或更新数据库密码
if ! gcloud secrets describe n8n-db-pass >/dev/null 2>&1; then
  echo -n "$NEON_PASS" | gcloud secrets create n8n-db-pass --data-file=-
else
  echo -n "$NEON_PASS" | gcloud secrets versions add n8n-db-pass --data-file=-
fi

# 写入或更新 Gmail App Password
if ! gcloud secrets describe n8n-smtp-pass >/dev/null 2>&1; then
  echo -n "$GMAIL_APP_PASS" | gcloud secrets create n8n-smtp-pass --data-file=-
else
  echo -n "$GMAIL_APP_PASS" | gcloud secrets versions add n8n-smtp-pass --data-file=-
fi

# 授予默认计算服务账号读取 Secret 的权限
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
SA_EMAIL="${PROJECT_NUMBER}-compute@developer.gserviceaccount.com"

for SECRET in n8n-encryption-key n8n-db-pass n8n-smtp-pass; do
  gcloud secrets add-iam-policy-binding $SECRET \
    --member="serviceAccount:${SA_EMAIL}" \
    --role="roles/secretmanager.secretAccessor" >/dev/null
done

echo "📦 [3/4] 部署 n8n 容器到 Google Cloud Run..."
gcloud run deploy $SERVICE_NAME \
  --image=n8nio/n8n:latest \
  --region=$REGION \
  --allow-unauthenticated \
  --port=5678 \
  --memory=2Gi \
  --cpu=1 \
  --min-instances=0 \
  --max-instances=2 \
  --no-cpu-throttling \
  --set-secrets="N8N_ENCRYPTION_KEY=n8n-encryption-key:latest,DB_POSTGRESDB_PASSWORD=n8n-db-pass:latest,N8N_SMTP_PASS=n8n-smtp-pass:latest" \
  --set-env-vars="N8N_PORT=5678,N8N_PROTOCOL=https,NODE_ENV=production" \
  --set-env-vars="GENERIC_TIMEZONE=Asia/Shanghai,TZ=Asia/Shanghai" \
  --set-env-vars="N8N_ENDPOINT_HEALTH=health" \
  --set-env-vars="DB_TYPE=postgresdb" \
  --set-env-vars="DB_POSTGRESDB_HOST=${NEON_HOST}" \
  --set-env-vars="DB_POSTGRESDB_PORT=5432" \
  --set-env-vars="DB_POSTGRESDB_DATABASE=${NEON_DB}" \
  --set-env-vars="DB_POSTGRESDB_USER=${NEON_USER}" \
  --set-env-vars="DB_POSTGRESDB_SSL=true" \
  --set-env-vars="DB_POSTGRESDB_SSL_REJECT_UNAUTHORIZED=false" \
  --set-env-vars="N8N_EMAIL_MODE=smtp" \
  --set-env-vars="N8N_SMTP_HOST=smtp.gmail.com" \
  --set-env-vars="N8N_SMTP_PORT=465" \
  --set-env-vars="N8N_SMTP_SSL=true" \
  --set-env-vars="N8N_SMTP_USER=${GMAIL_USER}" \
  --set-env-vars="N8N_SMTP_SENDER=${GMAIL_USER}"

echo "🔗 [4/4] 自动提取 Cloud Run 域名并回填 Webhook 环境变量..."
export SERVICE_URL=$(gcloud run services describe $SERVICE_NAME --region=$REGION --format='value(status.url)')
export SERVICE_HOST=${SERVICE_URL#https://}

gcloud run services update $SERVICE_NAME \
  --region=$REGION \
  --update-env-vars="WEBHOOK_URL=${SERVICE_URL}/,N8N_HOST=${SERVICE_HOST},N8N_EDITOR_BASE_URL=${SERVICE_URL}/"

echo "============================================================"
echo "🎉 部署大功告成！请点击以下链接访问并初始化你的 n8n："
echo "$SERVICE_URL"
echo "============================================================"
EOF
```

4. 使用编辑器（如 `nano deploy-n8n.sh`）修改脚本顶部的 7 个配置变量，保存后赋权并执行：

```bash
chmod +x deploy-n8n.sh
./deploy-n8n.sh
```

5. 等待约 1~2 分钟，点击终端最后输出的 `https://n8n-xxx.a.run.app` 链接，在网页中设置你的 n8n 管理员账号。

---

## 第四步：在 n8n 中添加 Gmail SMTP 凭证

登录 n8n 后台后，首先配置发信凭证供工作流调用：

1. 点击左侧导航栏的 **Credentials** -> 点击 **Add Credential**。
2. 搜索并选择 **SMTP**，点击 **Continue**。
3. 按照下表填写参数：

| 字段名称 | 填写内容 | 说明 |
| :--- | :--- | :--- |
| **User** | `你的邮箱@gmail.com` | 完整的 Gmail 地址 |
| **Password** | `16位应用专用密码` | 第二步生成的无空格 16 位字母（非登录密码） |
| **Host** | `smtp.gmail.com` | Gmail 官方 SMTP 服务器 |
| **Port** | `465` | SSL 安全端口（切勿填 25） |
| **SSL/TLS** | **开启（Switch ON）** | **必须打开**，与 465 端口配合使用 |

4. 点击右上角的 **Save** 按钮，看到顶部的绿色提示 `Connection tested successfully` 即代表配置成功。

---

## 第五步：导入“填写表单自动发邮件”工作流

n8n 原生自带高颜值的 **Form Trigger（表单触发器）**，无需借助任何第三方表单工具即可直接生成公开网页表单。

### 1. 复制工作流 JSON
复制以下完整的 JSON 代码：

```json
{
  "name": "表单提交自动发送邮件 (Cloud Run + Gmail SMTP)",
  "nodes": [
    {
      "parameters": {
        "formTitle": "获取专属资料 / 订阅通知",
        "formDescription": "请填写您的称呼与常用邮箱，提交后系统将在 10 秒内把准备好的完整资料发送至您的收件箱。",
        "formFields": {
          "values": [
            {
              "fieldLabel": "您的称呼",
              "placeholder": "例如：Alex",
              "requiredField": true
            },
            {
              "fieldLabel": "您的邮箱",
              "fieldType": "email",
              "placeholder": "name@example.com",
              "requiredField": true
            },
            {
              "fieldLabel": "您最感兴趣的内容（选填）",
              "fieldType": "dropdown",
              "fieldOptions": {
                "values": [
                  {
                    "option": "AI 与 Agent 工程实践"
                  },
                  {
                    "option": "开源项目与社区指南"
                  },
                  {
                    "option": "独立开发与增长实战"
                  }
                ]
              }
            }
          ]
        },
        "options": {
          "buttonLabel": "立即发送到我的邮箱",
          "respondWithOptions": {
            "values": {
              "respondWith": "text",
              "formSubmittedText": "🎉 提交成功！邮件已经发出，请前往您的收件箱查收（如未收到请留意垃圾箱）。"
            }
          }
        }
      },
      "id": "form-trigger-node",
      "name": "填写表单",
      "type": "n8n-nodes-base.formTrigger",
      "typeVersion": 2.2,
      "position": [
        260,
        300
      ],
      "webhookId": "exclusive-guide-form"
    },
    {
      "parameters": {
        "fromEmail": "Bestony <你的邮箱@gmail.com>",
        "toEmail": "={{ $json['您的邮箱'] }}",
        "subject": "=👋 你好 {{ $json['您的称呼'] }}，这是为你准备的专属资料",
        "emailFormat": "html",
        "html": "=<!DOCTYPE html>\n<html>\n<head>\n  <meta charset=\"utf-8\">\n</head>\n<body style=\"font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif; background-color: #f4f4f5; margin: 0; padding: 32px 16px;\">\n  <div style=\"max-width: 600px; margin: 0 auto; background: #ffffff; border-radius: 12px; padding: 32px; box-shadow: 0 2px 8px rgba(0,0,0,0.06);\">\n    <h2 style=\"color: #18181b; margin-top: 0;\">Hi {{ $json['您的称呼'] }}，</h2>\n    <p style=\"color: #3f3f46; font-size: 16px; line-height: 1.6;\">\n      感谢你填写表单！看到你对 <strong>{{ $json['您最感兴趣的内容（选填）'] || '我们的内容' }}</strong> 感兴趣，这封邮件里包含了我为你提前整理好的核心资料。\n    </p>\n    \n    <div style=\"background-color: #f8fafc; border-left: 4px solid #2563eb; padding: 16px; margin: 24px 0; border-radius: 4px;\">\n      <p style=\"margin: 0 0 8px 0; font-weight: 600; color: #1e293b;\">📦 核心资料清单：</p>\n      <ul style=\"margin: 0; padding-left: 20px; color: #475569; line-height: 1.8;\">\n        <li><strong>资料一</strong>：<a href=\"https://example.com\" style=\"color: #2563eb; text-decoration: none;\">点击查看在线文档 / 下载链接</a></li>\n        <li><strong>资料二</strong>：核心配置模板与避坑指南</li>\n      </ul>\n    </div>\n\n    <p style=\"color: #3f3f46; font-size: 16px; line-height: 1.6;\">\n      如果有任何问题，欢迎直接回复这封邮件与我交流！\n    </p>\n    \n    <hr style=\"border: none; border-top: 1px solid #e4e4e7; margin: 32px 0 16px 0;\">\n    <p style=\"color: #a1a1aa; font-size: 13px; margin: 0;\">\n      Best regards,<br>Bestony\n    </p>\n  </div>\n</body>\n</html>",
        "options": {}
      },
      "id": "send-email-node",
      "name": "发送预设邮件",
      "type": "n8n-nodes-base.emailSend",
      "typeVersion": 2.1,
      "position": [
        520,
        300
      ]
    }
  ],
  "connections": {
    "填写表单": {
      "main": [
        [
          {
            "node": "发送预设邮件",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

### 2. 粘贴并修改你的预设邮件内容
1. 在 n8n 中新建一个空白 **Workflow**，直接在画布空白处按 **`Ctrl + V`（Mac 按 `Cmd + V`）**，两个节点会自动生成。
2. 双击 **“填写表单”** 节点：
   * 可按需修改表单标题（`Form Title`）、描述（`Form Description`）以及表单字段。
   * *注意：如果你修改了 `Field Label`（例如把 `您的邮箱` 改为 `Email`），请同步修改下一个节点中的表达式引用。*
3. 双击 **“发送预设邮件”** 节点：
   * **Credential to connect with**: 选择第四步创建的 `SMTP account`。
   * **From Email**: 修改为 `你的名字 <你的邮箱@gmail.com>`。
   * **Subject & HTML**: 将里面的预设标题、正文和资料链接替换为你准备好的实际内容。

### 3. 测试与正式上线发布
1. **测试运行**：点击画布下方的 **Test workflow** 按钮，会自动弹出一个测试表单网页。填入你自己的邮箱并点击提交，确认在收件箱中收到格式精美的 HTML 邮件。
2. **激活上线**：点击画布右上角的开关，将工作流状态从 `Inactive` 切换为 **`Active`（绿色）**，并点击 **Save** 保存工作流。
3. **获取公开链接**：
   * 再次双击 **“填写表单”** 节点，展开顶部的 **Form URLs**。
   * 点击切换到 **Production URL**，复制该链接（格式如 `https://n8n-xxx.a.run.app/form/exclusive-guide-form`）。
   * 现在，你可以将这个链接直接分享给任何人、生成二维码，或者嵌入到你的文章与网站中！

---

## 第六步：日常运维与升级备忘

* **如何升级 n8n 版本？**
  因为所有数据都安全地保存在 Neon PostgreSQL 中，且加密密钥固定在 Secret Manager 里，未来升级 n8n 只需重新运行一次 `./deploy-n8n.sh`，Cloud Run 会拉取最新的 `n8nio/n8n:latest` 镜像并无缝切换，历史数据与工作流不会有任何丢失。
* **冷启动说明**：
  配置中设置了 `--min-instances=0` 以节省费用。当超过 15 分钟无人访问时，容器会自动休眠；当有新用户点击你的表单链接时，Cloud Run 会在 2~4 秒内瞬间冷启动唤醒容器并展示表单，后续提交与发信则在毫秒级完成。如果你追求极致的零等待体验，可将脚本中的 `--min-instances=0` 改为 `1`。
