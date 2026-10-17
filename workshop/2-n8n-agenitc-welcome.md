# GDG 现场智能提问墙 —— n8n × Gemini × 飞书 配置指南

> 参会者扫码提问 → Gemini 分类 + 初步解答 → 飞书卡片实时推送 → 按需发送定制资料邮件

---

## 为什么需要 LLM？

传统做法：用关键词匹配分类问题（Python 几行代码就能做）。

本方案 LLM 做的事远不止分类：

| LLM 输出字段 | 能力体现 | 价值 |
|---|---|---|
| `topic` | 理解语义，而非关键词匹配 | "我的 App 在 GKE 上跑不起来" → CLOUD_GCP |
| `urgency` | 判断问题的紧迫程度 | 部署故障 → high，原理探讨 → low |
| `summary` | 压缩提炼核心 | 长段描述 → 一句话摘要 |
| `tags` | 提取技术关键词 | ["Gemini", "流式输出", "Python"] |
| `question_type` | 识别问题意图 | 操作指南 / 排查问题 / 概念理解 / 观点讨论 |
| **`preliminary_answer`** | **直接给出初步解答** | **这是纯关键词匹配做不到的** |
| `reply_hint` | 个性化友好回应 | 引导参会者期待现场讨论 |

**`preliminary_answer` 是整个方案的核心价值点**：嘉宾打开飞书，看到的不只是"有人提问了"，而是 AI 已经帮他读懂了问题、给出了初步解答，他只需要在这个基础上补充或纠正。

---

## 架构一览

```
[参会者扫码]
     │
     ▼
[表单 Trigger]  ── 收集：称呼 / 问题 / 邮箱（选填）
     │
     ▼
[Code: 构建请求]  ── 拼装带 7 字段 Prompt 的 Gemini 请求体
     │
     ▼
[HTTP: Gemini 分析]  ── 返回 topic / urgency / summary / tags /
     │                   question_type / preliminary_answer / reply_hint
     ▼
[Code: 解析与映射]  ── 解析 JSON，生成飞书卡片（含 AI 解答）& 邮件 HTML
     │
     ▼
[HTTP: 飞书通知]  ── 同一个群，卡片颜色 + AI 解答 区分话题
     │
     ▼
[IF: 有邮箱?]
     ├── 是 ──► [Gmail: 发送资料邮件]（含 AI 解答 + 精选资料）
     └── 否 ──► 结束
```

### 话题 → 飞书卡片颜色对照

| 话题 | 卡片颜色 | 触发关键词（示例） |
|---|---|---|
| 🤖 AI & LLM | 紫色 | AI / Gemini / 大模型 / Agent / RAG |
| ☁️ Cloud & GCP | 蓝色 | GCP / Cloud Run / Firebase / K8s |
| 📱 Mobile & Flutter | 青色 | Flutter / Android / Dart / iOS |
| 🌐 Web & 前端 | 绿色 | Web / 前端 / React / Chrome / PWA |
| 💬 综合话题 | 灰色 | 其他所有问题 |

---

## 前置准备

在开始配置前，请确认以下条件均已满足：

- [x] n8n 已部署并可访问（参考 `cloud_run_n8n.md`）
- [x] Gmail SMTP 凭证已在 n8n 中配置（参考 `cloud_run_n8n.md` 第四步）
- [ ] 飞书自定义机器人 Webhook URL（Step 1 获取）
- [ ] Google Gemini API Key（Step 2 获取）

---

## Step 1：创建飞书机器人，获取 Webhook URL

1. 打开飞书，进入活动工作群（或新建专用群）。
2. 点击群右上角 **设置** → **机器人** → **添加机器人**。
3. 选择 **自定义机器人**，填写名称（如 `GDG 提问墙`），点击 **添加**。
4. 复制弹窗中的 **Webhook 地址**：
   ```
   https://open.feishu.cn/open-apis/bot/v2/hook/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
   ```
5. （可选）开启「自定义关键词」安全设置，填入 `GDG`，防止非法推送。
6. 点击 **完成**，保存 Webhook URL 备用。

---

## Step 2：获取 Google Gemini API Key

1. 访问 [Google AI Studio](https://aistudio.google.com/)，使用 Google 账号登录。
2. 点击左侧 **Get API key** → **Create API key**。
3. 选择 GCP 项目后点击 **Create API key in existing project**。
4. 复制生成的 API Key（格式：`AIzaSy...`），妥善保存备用。

> **费用说明**：Gemini 3.8 Flash 免费额度为每分钟 15 次;

---

## Step 3：导入 Workflow JSON

1. 登录 n8n 后台，新建空白 Workflow。
2. 在画布空白处按 **`Cmd + V`**（Windows 为 `Ctrl + V`），直接粘贴文末的[完整 JSON](#完整-workflow-json)。
3. 7 个节点会自动生成，连线已配置完毕。

---

## Step 4：逐节点配置

### 4.1 填写表单（Form Trigger）— 无需修改

双击节点，可选择性调整表单标题和引导语。

激活工作流后，在节点顶部的 **Form URLs** 中切换到 **Production URL**，生成二维码用于现场展示。

---

### 4.2 构建请求（Code Node）— 无需修改

自动将表单数据拼装成 Gemini API 请求体，Prompt 要求 Gemini 返回 7 个字段的 JSON。

---

### 4.3 Gemini 分析（HTTP Request）— 替换 API Key

双击节点，将 URL 中的 `YOUR_GEMINI_API_KEY` 替换为 Step 2 获取的密钥：

```
https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=你的APIKey
```

其余参数无需修改。

---

### 4.4 解析与映射（Code Node）— 无需修改

解析 Gemini 返回结果，生成：
- **飞书互动卡片**：含话题标签、紧迫度、问题类型、原始问题、**AI 初步解答**、技术标签
- **邮件 HTML**：含 AI 初步解答（蓝紫色高亮块）+ 精选资料链接

如需替换资料链接，找到代码中的 `rm` 对象，修改各话题 `res` 数组的 `t`（标题）和 `u`（链接）。

---

### 4.5 飞书通知（HTTP Request）— 替换 Webhook URL

双击节点，将 URL 替换为 Step 1 获取的飞书 Webhook 地址：

```
https://open.feishu.cn/open-apis/bot/v2/hook/你的Token
```

---

### 4.6 有邮箱?（IF Node）— 无需修改

参会者填写了邮箱 → true 分支 → 发送邮件；未填写 → 结束。

---

### 4.7 发送资料邮件（Email Send）— 配置凭证和发件人

| 字段 | 操作 |
|---|---|
| **Credential to connect with** | 选择已配置好的 SMTP 凭证 |
| **From Email** | 修改为 `你的名字 <你的邮箱@gmail.com>` |
| Subject / HTML | 由上游节点动态生成，**无需手动修改** |

---

## Step 5：测试与上线

### 功能测试

1. 点击画布下方 **Test workflow**，弹出测试表单页面。
2. 填入称呼、一个技术问题（如「如何用 Gemini API 做流式输出？」）和你自己的邮箱，提交。
3. 在 n8n 执行日志中依次检查：
   - **Gemini 分析**：查看返回 JSON，确认 7 个字段均存在，`preliminary_answer` 有实质内容
   - **飞书通知**：状态码为 `200`
   - **发送资料邮件**：确认已发出
4. 在飞书群中查看卡片是否正常渲染，「AI 初步解答」部分是否有内容。
5. 检查邮箱是否收到含「AI 初步解答」和「精选资料」两个区块的 HTML 邮件。

### 正式上线

1. 画布右上角开关切换为 **Active（绿色）**，点击 **Save**。
2. 双击「填写表单」→ **Form URLs** → 切换到 **Production URL** → 生成二维码投屏至现场。

---

## 飞书卡片效果（以 AI & LLM 话题为例）

```
┌─────────────────────────────────────────────────────┐
│ 🔔 新提问 · 🤖 AI & LLM                  [紫色标题]  │
├─────────────────────────────────────────────────────┤
│  提问者          │  紧迫度        │  类型            │
│  Alex           │  🔵 普通       │  🔧 操作指南      │
├─────────────────────────────────────────────────────┤
│  ❓ 问题内容                                         │
│  如何用 Gemini API 实现流式输出？                      │
├─────────────────────────────────────────────────────┤
│  🤖 AI 初步解答                                      │
│  Gemini API 通过 generateContentStream() 方法支持     │
│  流式输出，返回 AsyncGenerator，可逐 chunk 读取响应。   │
│  Python 示例：for chunk in model.generate_content_  │
│  stream(prompt): print(chunk.text, end="")          │
├─────────────────────────────────────────────────────┤
│  📝 Gemini 流式输出实现  |  🏷️ #Gemini #流式 #Python │
│  · GDG 智能提问墙 n8n × Gemini                       │
└─────────────────────────────────────────────────────┘
```

---

## 5 分钟现场 Demo 脚本

| 时间 | 动作 |
|---|---|
| 0:00–0:30 | 展示大屏二维码，邀请 1–2 名观众扫码提问 |
| 0:30–1:30 | 解说 n8n 画布，指出各节点职责（重点介绍 Gemini 那个节点） |
| 1:30–2:30 | 提问提交后，**实时展示 Gemini 返回的 JSON**，着重指出 `preliminary_answer` 字段 |
| 2:30–3:30 | 切换飞书群，展示卡片的颜色 + AI 初步解答 |
| 3:30–4:30 | 展示提问者收到的邮件（两个高亮区块：AI 解答 + 资料链接） |
| 4:30–5:00 | 总结：「传统关键词匹配做不到 preliminary_answer，这才是 LLM 的价值」 |

---

## 常见问题

**Q：Gemini 的 `preliminary_answer` 质量如何保证？**

Gemini 1.5 Flash 对常见技术问题（GCP、Flutter、Web 等）有良好的知识储备，回答质量通常可用。若需要更高准确率，可将模型改为 `gemini-1.5-pro`：将节点 URL 中的 `gemini-1.5-flash` 改为 `gemini-1.5-pro` 即可，响应时间约增加 1–2 秒。

**Q：飞书机器人收不到消息，状态码 200 但群里没有卡片？**

检查是否开启了「签名校验」。本方案未配置签名，若在机器人设置中开启，需在 HTTP Request 节点 Headers 中添加 `X-Lark-Signature` 签名头。建议测试阶段关闭签名校验。

**Q：Code 节点报 JSON parse 错误？**

节点有 `try/catch` 兜底，不会中断流程。若频繁出错，在 Gemini 节点 URL 后追加 `&alt=json`，或在 Prompt 中加强约束：在 `generationConfig` 中指定 `response_schema`。

**Q：想给高紧迫度问题单独 @ 嘉宾？**

在「解析与映射」Code 节点的 `feishuCard` elements 数组末尾，按 urgency 有条件追加 `at` 元素：
```javascript
...(cl.urgency === 'high' ? [{tag:'at', user_id:'ou_嘉宾飞书ID'}] : [])
```

---

## 完整 Workflow JSON

将以下 JSON 在 n8n 画布中按 `Cmd+V` 粘贴即可导入（需替换 2 处占位符）：

- `YOUR_GEMINI_API_KEY` → Step 2 获取的 Gemini API Key
- `YOUR_FEISHU_WEBHOOK_TOKEN` → Step 1 获取的飞书 Webhook 完整 URL

```json
{
  "name": "GDG 现场智能提问墙 (飞书 + Gemini)",
  "nodes": [
    {
      "parameters": {
        "formTitle": "🎤 GDG 现场提问",
        "formDescription": "提交你的问题，AI 将自动分析并推送给嘉宾，留下邮箱还可获取精选 Google 资料包！",
        "formFields": {
          "values": [
            { "fieldLabel": "你的称呼", "placeholder": "例如：Alex", "requiredField": true },
            { "fieldLabel": "你的问题", "fieldType": "textarea", "placeholder": "请输入你最想问的问题...", "requiredField": true },
            { "fieldLabel": "联系邮箱", "fieldType": "email", "placeholder": "name@example.com", "requiredField": false }
          ]
        },
        "options": {
          "buttonLabel": "提交问题",
          "respondWithOptions": {
            "values": {
              "respondWith": "text",
              "formSubmittedText": "🎉 已收到你的问题！AI 正在分析中，结果将发送至你的邮箱。"
            }
          }
        }
      },
      "id": "node-01",
      "name": "填写表单",
      "type": "n8n-nodes-base.formTrigger",
      "typeVersion": 2.2,
      "position": [180, 400],
      "webhookId": "gdg-ask-2024"
    },
    {
      "parameters": {
        "jsCode": "const fd=$input.first().json,q=fd['你的问题']||'',p='你是GDG活动现场的智能技术助手，请分析用户问题，仅返回如下JSON（不要任何其他文字）：\\n{\"topic\":\"AI_LLM\",\"urgency\":\"normal\",\"summary\":\"20字内摘要\",\"tags\":[\"标签1\",\"标签2\"],\"question_type\":\"how-to\",\"preliminary_answer\":\"100字内初步解答\",\"reply_hint\":\"30字内友好回应\"}\\ntopic枚举(仅选一)：AI_LLM(涉及AI/Gemini/大模型/Agent/RAG/Prompt)|CLOUD_GCP(涉及GCP/CloudRun/Firebase/Kubernetes/云架构)|MOBILE_FLUTTER(涉及Flutter/Android/Dart/iOS/移动端)|WEB_FRONTEND(涉及Web/前端/React/Chrome/PWA)|OTHER(其他所有话题)\\nurgency：high(部署故障或阻塞性问题)/normal(普通技术问题)/low(概念性讨论或闲聊)\\ntags：提取2-4个关键技术词的数组\\nquestion_type：how-to(如何实现)/debug(排查问题)/concept(概念理解)/opinion(观点讨论)/other\\npreliminary_answer：基于你的技术知识给出简洁准确的初步解答，控制在100字内；若问题太模糊则填写\"需要更多上下文才能准确解答\"\\nreply_hint：给提问者的一句话友好回应，引导他期待现场讨论\\n\\n用户问题：'+q;return [{json:{...fd,geminiBody:JSON.stringify({contents:[{parts:[{text:p}]}],generationConfig:{response_mime_type:'application/json'}}),_name:fd['你的称呼']||'朋友',_q:q,_email:fd['联系邮箱']||''}}];"
      },
      "id": "node-02",
      "name": "构建请求",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [420, 400]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://generativelanguage.googleapis.com/v1beta/models/gemini-3.8-flash:generateContent?key=YOUR_GEMINI_API_KEY",
        "authentication": "none",
        "sendBody": true,
        "contentType": "raw",
        "rawContentType": "application/json",
        "body": "={{ $json.geminiBody }}",
        "options": {}
      },
      "id": "node-03",
      "name": "Gemini 分析",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [660, 400]
    },
    {
      "parameters": {
        "jsCode": "const input=$input.first().json,raw=input.candidates?.[0]?.content?.parts?.[0]?.text||'{}';\nlet cl;try{cl=JSON.parse(raw);}catch(e){cl={topic:'OTHER',urgency:'normal',summary:(input._q||'').slice(0,20),tags:[],question_type:'other',preliminary_answer:'需要更多上下文才能准确解答',reply_hint:'感谢你的提问！'};}\nconst rm={AI_LLM:{label:'🤖 AI & LLM',color:'purple',res:[{t:'Google AI Studio — 免费体验 Gemini API',u:'https://aistudio.google.com/'},{t:'Gemini API 官方文档',u:'https://ai.google.dev/gemini-api/docs'},{t:'Vertex AI 快速入门 Codelab',u:'https://codelabs.developers.google.com/codelabs/gemini-vertex-ai-tour'}]},CLOUD_GCP:{label:'☁️ Cloud & GCP',color:'blue',res:[{t:'Google Cloud Skills Boost — 免费学习路径',u:'https://cloudskillsboost.google/'},{t:'Cloud Run 官方文档',u:'https://cloud.google.com/run/docs'},{t:'GCP 架构中心 — 最佳实践参考',u:'https://cloud.google.com/architecture'}]},MOBILE_FLUTTER:{label:'📱 Mobile & Flutter',color:'turquoise',res:[{t:'Flutter 官方文档',u:'https://docs.flutter.dev/'},{t:'Firebase 快速入门指南',u:'https://firebase.google.com/docs/guides'},{t:'Flutter Codelabs 合集',u:'https://docs.flutter.dev/codelabs'}]},WEB_FRONTEND:{label:'🌐 Web & 前端',color:'green',res:[{t:'web.dev 官方学习路径',u:'https://web.dev/learn'},{t:'Chrome for Developers',u:'https://developer.chrome.com/'},{t:'PageSpeed Insights — 性能测速工具',u:'https://pagespeed.web.dev/'}]},OTHER:{label:'💬 综合话题',color:'grey',res:[{t:'Google Codelabs — 动手实验合集',u:'https://codelabs.developers.google.com/'},{t:'GDG 社区主页',u:'https://developers.google.com/community/gdg'},{t:'Google Developers 官方博客',u:'https://developers.googleblog.com/'}]}};\nconst td=rm[cl.topic]||rm.OTHER,name=input._name||'朋友',q=input._q||'',email=input._email||'';\nconst urgMap={high:'⚠️ 紧急',normal:'🔵 普通',low:'🔘 低'};\nconst qtMap={'how-to':'🔧 操作指南',debug:'🐛 问题排查',concept:'💡 概念理解',opinion:'💬 观点讨论',other:'❓ 其他'};\nconst tags=Array.isArray(cl.tags)?cl.tags.map(t=>'#'+t).join('  '):''\nconst qtLabel=qtMap[cl.question_type]||'❓ 其他';\nconst pa=cl.preliminary_answer||'暂无初步解答';\nconst feishuCard=JSON.stringify({msg_type:'interactive',card:{header:{title:{content:'🔔 新提问 · '+td.label,tag:'plain_text'},template:td.color},elements:[{tag:'div',fields:[{is_short:true,text:{tag:'lark_md',content:'**提问者**\\n'+name}},{is_short:true,text:{tag:'lark_md',content:'**紧迫度**\\n'+(urgMap[cl.urgency]||'🔵 普通')}},{is_short:true,text:{tag:'lark_md',content:'**类型**\\n'+qtLabel}}]},{tag:'div',text:{tag:'lark_md',content:'**❓ 问题内容**\\n'+q}},{tag:'hr'},{tag:'div',text:{tag:'lark_md',content:'**🤖 AI 初步解答**\\n'+pa}},{tag:'hr'},{tag:'note',elements:[{tag:'plain_text',content:'📝 '+cl.summary+(tags?'  |  🏷️ '+tags:'')+'  ·  GDG 智能提问墙 n8n × Gemini'}]}]}});\nconst rl=td.res.map(r=>`<li><a href=\"${r.u}\" style=\"color:#1a73e8;text-decoration:none;\">${r.t}</a></li>`).join('');\nconst emailHtml=`<!DOCTYPE html><html><head><meta charset='utf-8'></head><body style='font-family:-apple-system,BlinkMacSystemFont,Segoe UI,Roboto,sans-serif;background:#f5f5f5;margin:0;padding:32px 16px;'><div style='max-width:600px;margin:0 auto;background:#fff;border-radius:12px;padding:32px;box-shadow:0 2px 8px rgba(0,0,0,.06);'><h2 style='color:#1a1a1a;margin-top:0;'>Hi ${name}，</h2><p style='color:#444;font-size:16px;line-height:1.6;'>感谢你在 <strong>GDG 活动现场</strong>提问！🎉 你的问题已归类到 <strong>${td.label}</strong>，Gemini 已为你生成了初步解答：</p><div style='background:#f0f4ff;border-left:4px solid #4f46e5;padding:16px;margin:20px 0;border-radius:4px;'><p style='margin:0 0 8px;font-weight:600;color:#1a1a1a;'>🤖 AI 初步解答</p><p style='margin:0;color:#444;font-size:15px;line-height:1.7;'>${pa}</p></div><div style='background:#f8fafc;border-left:4px solid #1a73e8;padding:16px;margin:20px 0;border-radius:4px;'><p style='margin:0 0 8px;font-weight:600;color:#1a1a1a;'>📦 精选学习资料</p><ul style='margin:0;padding-left:20px;line-height:2;'>${rl}</ul></div><p style='font-style:italic;background:#fafafa;padding:12px 16px;border-radius:8px;color:#555;'>「${cl.reply_hint}」</p><p style='color:#444;font-size:16px;line-height:1.6;'>欢迎加入 GDG 社区，与更多开发者保持连接！</p><hr style='border:none;border-top:1px solid #e4e4e7;margin:20px 0 12px;'><p style='color:#999;font-size:13px;margin:0;'>Best regards,<br>GDG 活动团队 · Powered by n8n × Gemini</p></div></body></html>`;\nreturn [{json:{...input,topic:cl.topic,urgency:cl.urgency,summary:cl.summary,tags:cl.tags,question_type:cl.question_type,preliminary_answer:pa,reply_hint:cl.reply_hint,topicLabel:td.label,feishuCard,emailHtml,hasEmail:!!(email&&email.trim())}}];"
      },
      "id": "node-04",
      "name": "解析与映射",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [900, 400]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://open.feishu.cn/open-apis/bot/v2/hook/YOUR_FEISHU_WEBHOOK_TOKEN",
        "authentication": "none",
        "sendBody": true,
        "contentType": "raw",
        "rawContentType": "application/json",
        "body": "={{ $json.feishuCard }}",
        "options": {}
      },
      "id": "node-05",
      "name": "飞书通知",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [1140, 400]
    },
    {
      "parameters": {
        "conditions": {
          "options": { "caseSensitive": true, "leftValue": "", "typeValidation": "strict", "version": 2 },
          "conditions": [{ "id": "email-check", "leftValue": "={{ $json.hasEmail }}", "rightValue": "", "operator": { "type": "boolean", "operation": "true", "singleValue": true } }],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "node-06",
      "name": "有邮箱?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [1380, 400]
    },
    {
      "parameters": {
        "fromEmail": "GDG 活动团队 <你的邮箱@gmail.com>",
        "toEmail": "={{ $json._email }}",
        "subject": "=🤖 {{ $json._name }}，你的提问「{{ $json.summary }}」— AI 解答 + 专属资料",
        "emailFormat": "html",
        "html": "={{ $json.emailHtml }}",
        "options": {}
      },
      "id": "node-07",
      "name": "发送资料邮件",
      "type": "n8n-nodes-base.emailSend",
      "typeVersion": 2.1,
      "position": [1620, 280]
    }
  ],
  "connections": {
    "填写表单":   { "main": [[{ "node": "构建请求",   "type": "main", "index": 0 }]] },
    "构建请求":   { "main": [[{ "node": "Gemini 分析", "type": "main", "index": 0 }]] },
    "Gemini 分析": { "main": [[{ "node": "解析与映射", "type": "main", "index": 0 }]] },
    "解析与映射": { "main": [[{ "node": "飞书通知",   "type": "main", "index": 0 }]] },
    "飞书通知":   { "main": [[{ "node": "有邮箱?",    "type": "main", "index": 0 }]] },
    "有邮箱?": {
      "main": [
        [{ "node": "发送资料邮件", "type": "main", "index": 0 }],
        []
      ]
    }
  },
  "pinData": {},
  "settings": { "executionOrder": "v1" }
}
```
