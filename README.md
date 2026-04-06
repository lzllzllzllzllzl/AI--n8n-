# 🤖 AI新闻每日采集 + 飞书推送 — n8n 自动化工作流

> 每天早上7点自动采集4个AI媒体的最新文章，用大模型结构化提取关键信息，写入飞书多维表格，再由 AI Agent 汇总生成日报，一键推送飞书群。全程零人工干预。

---

## 效果预览

| 功能 | 描述 |
|------|------|
| 📰 采集来源 | 新智元、36氪、腾讯科技、量子位（4路并行） |
| 🧠 AI处理 | 豆包大模型提取标题/日期/摘要/媒体/链接 |
| 📊 数据存储 | 写入飞书多维表格（含日期、创建时间、链接跳转） |
| 💬 消息推送 | AI Agent 生成日报，推送到飞书群机器人 |
| ⏰ 触发方式 | 每天早上7点定时自动执行 |

---
<img width="2335" height="903" alt="image" src="https://github.com/user-attachments/assets/bd7b0d1b-2f6c-4962-98ba-e9ad599b8e3f" />


## 目录

- [一、需要准备什么](#一需要准备什么)
- [二、整体工作流架构](#二整体工作流架构)
- [三、手把手配置教程](#三手把手配置教程)
  - [Step 1 安装 n8n](#step-1-安装-n8n)
  - [Step 2 安装飞书插件](#step-2-安装飞书插件)
  - [Step 3 配置豆包凭证](#step-3-配置豆包凭证)
  - [Step 4 配置飞书凭证](#step-4-配置飞书凭证)
  - [Step 5 创建飞书多维表格](#step-5-创建飞书多维表格)
  - [Step 6 导入工作流](#step-6-导入工作流)
  - [Step 7 填入你自己的配置](#step-7-填入你自己的配置)
  - [Step 8 测试运行](#step-8-测试运行)
- [四、节点详解](#四节点详解)
- [五、常见报错和解决办法](#五常见报错和解决办法)
- [六、进阶定制](#六进阶定制)

---

## 一、需要准备什么

### 必须有的账号
| 工具 | 用途 | 费用 |
|------|------|------|
| [n8n](https://n8n.io) | 自动化工作流平台 | 自部署免费 |
| [火山引擎](https://console.volcengine.com/ark) | 豆包大模型 API | 按量付费，极低 |
| 飞书 | 多维表格 + 群机器人 | 免费 |

### 本地运行 n8n 的环境要求
- 安装 [Node.js](https://nodejs.org/) 18+ 版本
- 或者安装 [Docker](https://www.docker.com/)（推荐，更简单）

---

## 二、整体工作流架构

```
定时触发(每天7点)
        │
        ├──── 新智元RSS ──── 整理字段 ──── 限制5条 ──── LLM结构化 ──── 解析JSON ──── 写入飞书 ──┐
        │                                              │(豆包模型)                              │
        ├──── 36氪RSS ────── 整理字段 ──── 限制5条 ──── LLM结构化 ──── 解析JSON ──── 写入飞书 ──┤
        │                                              │(豆包模型)                   合并-新智元-36氪
        ├──── 腾讯科技RSS ── 整理字段 ──── 限制5条 ──── LLM结构化 ──── 解析JSON ──── 写入飞书 ──┤
        │                                              │(豆包模型)                   合并-腾讯-量子位
        └──── 量子位RSS ──── 整理字段 ──── 限制5条 ──── LLM结构化 ──── 解析JSON ──── 写入飞书 ──┘
                                                                                          │
                                                                                     最终合并(等4路全部完成)
                                                                                          │
                                                                                   AI Agent汇总推送
                                                                                   (豆包模型 + 读取飞书工具)
                                                                                          │
                                                                                    发送飞书群消息
```

**关键设计：**
- 4路RSS **完全并行**，互不阻塞，大幅节省时间
- 3个 Merge 节点串联（`合并-新智元-36氪` → `合并-腾讯-量子位` → `最终合并`），确保4路**全部写完飞书**后，才触发一次 Agent，只发一条群消息

---

## 三、手把手配置教程

### Step 1 安装 n8n

**方式A：Docker（推荐小白）**

```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

浏览器打开 `http://localhost:5678`，注册账号即可。

**方式B：npm**

```bash
npm install n8n -g
n8n start
```

---

### Step 2 安装飞书插件

n8n 默认没有飞书节点，需要安装社区插件：

1. 打开 n8n → 左下角头像 → **Settings**
2. 左侧选 **Community nodes**
3. 点击 **Install a community node**
4. 输入：`n8n-nodes-feishu-lite`
5. 点击 **Install**，等待安装完成

> ⚠️ 安装后需要**重启 n8n** 才能看到飞书节点

---

### Step 3 配置豆包凭证

豆包 API 是兼容 OpenAI 格式的，在 n8n 里当作 OpenAI 来配置。

**第一步：获取 API Key**
1. 打开 [火山引擎 - 方舟平台](https://console.volcengine.com/ark)
2. 左侧菜单 → **API Key 管理** → 创建 API Key，复制保存
3. 左侧菜单 → **模型广场** → 找到 `doubao-seed-1-6-lite`，开通并记录模型 ID

**第二步：在 n8n 添加凭证**
1. n8n 左侧 → **Credentials** → **Add credential**
2. 搜索选择 `OpenAI`
3. 填写：
   - **API Key**：填你的火山引擎 API Key
   - **Base URL**：`https://ark.cn-beijing.volces.com/api/v3`
4. 保存

---

### Step 4 配置飞书凭证

需要先在飞书开放平台创建应用获取权限。

**第一步：创建飞书应用**
1. 打开 [飞书开放平台](https://open.feishu.cn/)
2. 控制台 → **创建企业自建应用**
3. 填写应用名称（随意）→ 创建
4. 左侧 → **凭证与基础信息** → 记录 `App ID` 和 `App Secret`

**第二步：开通权限**
左侧 → **权限管理** → 搜索并开通以下权限：
- `bitable:app`（多维表格读写）
- `bitable:app:readonly`（读取）

**第三步：发布应用**
左侧 → **版本管理与发布** → 创建版本 → 申请发布（企业内部应用审核较快）

**第四步：在 n8n 添加凭证**
1. n8n → **Credentials** → **Add credential**
2. 搜索 `Feishu`，选择 `Feishu OAuth2 API`
3. 填入 App ID 和 App Secret
4. 点击 **Connect** 完成 OAuth 授权

---

### Step 5 创建飞书多维表格

1. 飞书 → 新建多维表格
2. 创建以下字段（字段名和类型必须对应）：

| 字段名 | 类型 |
|--------|------|
| 标题 | 文本 |
| 日期 | 日期 |
| 创建时间 | 日期 |
| 内容 | 文本 |
| 媒体 | 文本 |
| 链接 | 超链接 |

3. 打开表格 URL，从地址栏复制两个 ID：
   - `app_token`：URL 中 `/base/` 后面那段，例如 `Zhm6wRjKHiBJCokPoYBcyEK6nNf`
   - `table_id`：URL 中 `?table=` 后面那段，例如 `tbl3bbC4kxM8hHCe`

---

### Step 6 导入工作流

1. 下载本仓库的 `workflow.json` 文件
2. n8n 首页 → 右上角 **+** → **Import from file**
3. 选择 `workflow.json` 导入

---

### Step 7 填入你自己的配置

导入后需要替换以下内容（在 n8n 画布上逐个点击节点修改）：

**① 所有豆包模型节点（共5个）**
- 点击 `豆包模型(结构化)` / `豆包模型-36氪` / `豆包模型-腾讯` / `豆包模型-量子位` / `豆包模型(Agent)`
- Credential 选择你在 Step 3 创建的凭证
- Model ID 填：`doubao-seed-1-6-lite-251015`（或你开通的模型 ID）

**② 所有飞书写入节点（共4个）**
- 点击 `写入飞书多维表格` / `写入飞书-36氪` / `写入飞书-腾讯` / `写入飞书-量子位`
- Credential 选择你在 Step 4 创建的凭证
- `app_toke` 字段填你的 `app_token`
- `table_id` 字段填你的 `table_id`

**③ 读取飞书多维表格(工具) 节点**
- 同上，填入 app_token 和 table_id

**④ 发送飞书群消息 节点**
- 在飞书群 → 群设置 → 机器人 → 添加自定义机器人 → 复制 Webhook 地址
- 将节点里的 URL 替换为你的 Webhook 地址

---

### Step 8 测试运行

1. 点击画布右上角 **Test workflow** 按钮
2. 观察每个节点是否变绿（成功）
3. 检查飞书多维表格是否新增了记录
4. 检查飞书群是否收到了消息

测试通过后，点击右上角开关 **Active** 激活工作流，每天7点将自动执行。

---

## 四、节点详解

### 定时触发节点
设置为每天 07:00 触发，可在节点里修改时间。

### RSS采集节点（4个）
| 媒体 | RSS地址 |
|------|---------|
| 新智元 | `https://plink.anyfeeder.com/weixin/AI_era`（需开启 ignoreSSL） |
| 36氪 | `https://36kr.com/feed` |
| 腾讯科技 | `https://plink.anyfeeder.com/weixin/qqtech` |
| 量子位 | `https://www.qbitai.com/feed` |

### 整理字段节点
把 RSS 原始字段（`title`、`pubDate`、`link`、`description` 等）统一映射为：`标题`、`日期`、`链接`、`内容`、`媒体`，方便后续处理。

### 限制条数节点
每路只取最新 5 条，避免 API 调用过多。可自行修改数量。

### LLM结构化节点（chainLlm）
给大模型的提示词要求它只输出 JSON，包含5个字段：

```
{
  "标题": "...",
  "日期": "YYYY-MM-DD",
  "内容": "100字以内摘要",
  "媒体": "...",
  "链接": "..."
}
```

### 解析JSON节点（Code节点）
大模型偶尔会在 JSON 外面加 ```json 代码块，这里做容错处理：

```javascript
const raw = $input.item.json.text || '';
let parsed = {};
try {
  // 先清理 markdown 代码块标记
  const cleaned = raw.replace(/```json\s*/gi, '').replace(/```\s*/g, '').trim();
  parsed = JSON.parse(cleaned);
} catch(e) {
  // 兜底：正则提取 {} 范围内的内容
  const match = raw.match(/\{[\s\S]*\}/);
  if (match) { try { parsed = JSON.parse(match[0]); } catch(e2) {} }
}
// 日期转毫秒时间戳（飞书日期字段需要）
const dateStr = String(parsed['日期'] || '');
let dateTs = Date.now();
if (dateStr) { const d = new Date(dateStr); if (!isNaN(d.getTime())) dateTs = d.getTime(); }

return { json: {
  '标题': String(parsed['标题'] || '无标题'),
  '日期_ts': dateTs,
  '采集时间': Date.now(),
  '内容': String(parsed['内容'] || ''),
  '媒体': String(parsed['媒体'] || '未知媒体'),
  '链接': String(parsed['链接'] || '')
}};
```

### 写入飞书节点
body 用 `JSON.stringify()` 整体序列化，避免飞书 API 解析问题：

```javascript
={{ JSON.stringify({ fields: {
  "标题": $json['标题'],
  "日期": $json['日期_ts'],
  "创建时间": $json['采集时间'],
  "内容": $json['内容'],
  "媒体": $json['媒体'],
  "链接": { text: "查看原文", link: $json['链接'] }
} }) }}
```

> **注意**：飞书日期字段存的是**毫秒时间戳**（数字），不是字符串。

### Merge节点（3个串联）
用于等待所有并行分支完成：
- `合并-新智元-36氪`：等新智元路 + 36氪路都写完飞书
- `合并-腾讯-量子位`：等腾讯路 + 量子位路都写完飞书
- `最终合并`：等前两组都完成 → 触发 AI Agent

模式选 **Append**，将两路数据合并为一个列表向下传递。

### AI Agent节点
提示词让它调用 `news` 工具（读取飞书多维表格）获取今日新闻，然后格式化输出 Markdown 日报。

---

## 五、常见报错和解决办法

**Q：`A Model sub-node must be connected and enabled`**
> LLM 节点没有连接模型子节点。检查豆包模型节点是否通过 `ai_languageModel` 连接线连到了对应的 LLM 节点（连接类型不能是普通的 main 线）。

**Q：飞书日期显示 1970/01/21**
> 写入的日期值是秒级时间戳，但飞书需要毫秒级。确认 Code 节点里用的是 `d.getTime()`（返回毫秒）而不是 `/ 1000`。

**Q：飞书链接字段显示"查看原文"但点击无效**
> body 没有用 `JSON.stringify()` 包裹，导致嵌套表达式取值失败，链接变成空字符串。

**Q：每次只发一条飞书消息但只包含一路内容**
> Merge 节点的两个输入口没有分别连接两条线。检查 input 0 和 input 1 是否各连了一路（不能两条都连到 input 0）。

**Q：新智元的链接是搜狗搜索页**
> 这是微信公众号 RSS 代理的固有限制，搜狗链接点进去可以跳转到文章，目前暂无免费方案绕过。

---

## 六、进阶定制

- **修改采集数量**：找到各路的"限制条数"节点，修改 `maxItems` 值
- **修改触发时间**：点击"早晨定时触发"节点，修改小时数
- **新增媒体来源**：复制一路完整链路（RSS→整理→限制→LLM→Code→飞书写入），连接到对应 Merge 节点
- **修改 AI 日报风格**：在"AI Agent汇总推送"节点的 System Prompt 里修改输出格式

---

## 技术栈

| 组件 | 版本/说明 |
|------|---------|
| n8n | v1.x，执行模式 v1 |
| 豆包模型 | doubao-seed-1-6-lite-251015 |
| 飞书插件 | n8n-nodes-feishu-lite |
| LLM节点 | chainLlm v1.9 + lmChatOpenAi v1.3 |

---

*如有问题欢迎提 Issue。*
