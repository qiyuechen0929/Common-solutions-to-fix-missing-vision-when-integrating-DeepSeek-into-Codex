# Codex 视觉桥：图片识别方法全攻略

给纯文本推理模型（Codex、Claude Code、DeepSeek 等）补上"眼睛"的多通道视觉增强方案。

核心思路只有一句：**把图片变成文字/JSON，再让主模型推理。**

> 核验日期：2026-08-08。所有免费额度说明以各平台官方控制台为准。

---

## 目录

1. [为什么需要视觉桥](#为什么需要视觉桥)
2. [方法总览](#方法总览)
3. [方法详解](#方法详解)
4. [免费视觉模型推荐](#免费视觉模型推荐)
5. [配置教程](#配置教程)
6. [Skill 位置与安装](#skill-位置与安装)
7. [快速开始](#快速开始)
8. [输出规范](#输出规范)
9. [路由与降级链](#路由与降级链)
10. [缓存与成本优化](#缓存与成本优化)
11. [隐私说明](#隐私说明)
12. [跨工具使用](#跨工具使用)
13. [常见问题排查](#常见问题排查)
14. [验收清单](#验收清单)
15. [许可证](#许可证)

---

## 为什么需要视觉桥

桌面版聊天工具粘贴图片时，模型通常拿不到图片像素，拿到的只是系统临时文件路径，例如：

```text
C:\Users\26529\AppData\Local\Temp\codex-clipboard-xxxx.png
```

两条出路：

- 模型本身支持图像输入，直接读取该文件（零配置，效果最好）
- 模型不支持图像输入，就调用外部视觉 API 或本地 OCR，先把图片转成结构化文字，再交给主模型

后者就是"视觉桥"。

---

## 方法总览

| 方法 | 类型 | 需要网络 | 需要 Key | 成本 | 适合场景 |
|---|---|---|---|---|---|
| Codex 原生 `view_image` | 多模态直读 | 否 | 否 | 模型额度 | 模型支持图像时最优先 |
| GLM-4V-Flash | 云端视觉理解 | 是 | `GLM_API_KEY` | 免费 | 通用描述、简单问答 |
| GLM-4.1V-Thinking-Flash | 云端视觉推理 | 是 | `GLM_API_KEY` | 免费 | 图表、数学题、代码截图、复杂推理 |
| MinerU | 文档解析 | 是 | flash 免 token；extract 需 `MINERU_TOKEN` | flash 免费额度 | PDF、论文、表格、公式 |
| 百度 OCR | OCR | 是 | `BAIDU_API_KEY` + `BAIDU_SECRET_KEY` | 免费额度 | 截图文字、票据、扫描件 |
| Windows OCR | OCR | 否 | 否 | 系统自带 | 离线文字兜底 |
| PaddleOCR / Tesseract | OCR | 否 | 否 | 免费开源 | 本地离线、批量 OCR |
| Ollama / LM Studio / llama.cpp | 本地视觉模型 | 否 | 否 | 免费开源 | 隐私敏感、完全离线 |

---

## 方法详解

### 1. Codex 原生多模态直读

Codex 桌面版内置 `view_image` 工具，可以查看本地图片。

```text
用户发图片 -> 得到本地绝对路径 -> view_image 读取 -> 模型直接理解
```

优点：

- 零配置、零 Key、零额外成本
- 模型本身的视觉理解能力最强，适合复杂推理
- 不需要压缩、编码、网络请求

缺点：

- 依赖当前模型是否支持图像输入
- 当前会话如果模型不支持图像，工具拿到图片也可能看不到内容

### 2. GLM 视觉理解（推荐主力）

智谱 AI 的 OpenAI 兼容接口，两个免费模型：

| 通道 | 模型 | 定位 |
|---|---|---|
| `glm` | `glm-4v-flash` | 简单任务，快、免费 |
| `glm-thinking` | `glm-4.1v-thinking-flash` | 复杂推理，更强 |

接口信息：

```text
POST https://open.bigmodel.cn/api/paas/v4/chat/completions
Authorization: Bearer <GLM_API_KEY>
```

图片以 base64 放进 `image_url` 字段。

优点：

- 免费，官方渠道，2026-08-01 本机实测可用
- `glm` 与 `glm-thinking` 共用一个 `GLM_API_KEY`
- 中文理解好，混合图文识别稳定
- OpenAI 兼容协议，脚本和生态成熟

缺点：

- 需要联网
- 图片会发送到智谱服务器，隐私敏感场景不适用
- 免费额度可能随官方政策变化

### 3. MinerU 文档解析

适合 PDF、论文、报告、表格、公式，输出 Markdown。

```powershell
scripts\mineru-extract.ps1 -FilePath <文档路径> -Mode flash -Json
scripts\mineru-extract.ps1 -FilePath <文档路径> -Mode extract -Json
```

已知限制（2026-08-01 复核）：

- `flash` 模式无需 token，免费，约 10MB / 20 页
- `extract` 模式需要 `MINERU_TOKEN`

优点：

- 表格、版式、公式恢复能力远超普通 OCR
- 长文档不用逐页喂图

缺点：

- 不适合单张自然图片的"看懂"
- 需要安装 `mineru-open-api`

### 4. OCR 三件套

#### 百度 OCR

```powershell
scripts\baidu-ocr.ps1 -ImagePath <图片路径> -Json          # 标准版
scripts\baidu-ocr.ps1 -ImagePath <图片路径> -Accurate -Json # 高精度版
```

- 中英文混合效果好
- 有免费额度，需要 `BAIDU_API_KEY` + `BAIDU_SECRET_KEY`
- 需要联网

#### Windows OCR

```powershell
scripts\windows-ocr.ps1 -ImagePath <图片路径> -Json
```

- 系统自带，离线，无需 Key
- 适合断网兜底和隐私场景
- 复杂版式、手写、低清图质量一般

#### PaddleOCR / Tesseract（可选）

- 完全本地、开源免费
- 适合批量离线 OCR
- 需要自己安装和调参

### 5. 本地视觉模型

通过 Ollama / LM Studio / llama.cpp 起一个本地 OpenAI 兼容服务：

| 运行时 | 端口 | 说明 |
|---|---|---|
| Ollama | 11434 | 首选，`ollama pull qwen2.5-vl:3b` |
| LM Studio | 1234 | 图形界面，起本地服务 |
| llama.cpp | 8080 | `llama-server -m model.gguf --port 8080` |

推荐模型：

- VRAM >= 8GB：`qwen2.5-vl:7b`、`llama3.2-vision:11b`
- VRAM >= 4GB：`qwen2.5-vl:3b`、`minicpm-v`、`moondream`
- 无 GPU：`moondream`、`smolvlm`（CPU 可跑，较慢）
- 纯 OCR 场景候选：`deepseek-ai/DeepSeek-OCR-2`

优点：

- 完全离线、免费、隐私安全
- 数据不出本机

缺点：

- 需要硬件支持，小模型识别能力弱于云端大模型
- 首次下载模型体积较大

---

## 免费视觉模型推荐

| 模型 | 服务商 | 免费情况 | 获取/注册 | 适合场景 |
|---|---|---|---|---|
| GLM-4V-Flash | 智谱 AI | 永久免费（官方） | [open.bigmodel.cn](https://open.bigmodel.cn/) | 通用图片描述、简单问答 |
| GLM-4.1V-Thinking-Flash | 智谱 AI | 免费（官方） | 同上，共用 Key | 图表分析、数学题、代码截图、复杂推理 |
| Qwen2.5-VL 3B/7B | 阿里开源 | 本地免费 | `ollama pull qwen2.5-vl:3b` | 离线通用识别 |
| llama3.2-vision:11b | Meta 开源 | 本地免费 | Ollama | 高配机器离线识别 |
| minicpm-v | 开源 | 本地免费 | Ollama | 低显存 |
| moondream | 开源 | 本地免费 | Ollama | 无 GPU / CPU |
| smolvlm | 开源 | 本地免费 | Ollama | 极小模型、CPU |
| DeepSeek-OCR-2 | DeepSeek 开源 | 本地免费 | llmfit 选型 | 纯 OCR |
| MinerU flash | MinerU | flash 免费额度 | [MinerU 官网](https://mineru.net/) | PDF、表格、公式 |
| 百度 OCR | 百度智能云 | 免费额度 | [控制台](https://console.bce.baidu.com/ai/#/ai/ocr/app/list) | 中英文截图、票据 |
| Windows OCR | 微软 | 系统自带 | 无需注册 | 离线文字兜底 |

优先级建议：

```text
有网 + 简单图        -> GLM-4V-Flash
有网 + 复杂图/截图   -> GLM-4.1V-Thinking-Flash
有网 + PDF/表格/公式 -> MinerU flash
有网 + 只要文字      -> 百度 OCR
无网 + 要隐私        -> Windows OCR -> 本地 Ollama 模型
```

---

## 配置教程

### 1. 智谱 GLM（推荐第一个配）

注册地址：[https://open.bigmodel.cn/](https://open.bigmodel.cn/)

步骤：

1. 注册账号
2. 进入控制台，创建 API Key
3. 配置到本机（会写入用户级环境变量并验证）：

```powershell
scripts\setup.ps1 -SetKey -Channel glm -Key <你的key> -Verify
```

`glm-thinking` 与 `glm` 共用同一个 `GLM_API_KEY`，无需重复配置。

### 2. 百度 OCR

注册地址：[https://console.bce.baidu.com/ai/#/ai/ocr/app/list](https://console.bce.baidu.com/ai/#/ai/ocr/app/list)

步骤：

1. 登录百度智能云，创建 OCR 应用
2. 拿到 API Key 和 Secret Key
3. 配置：

```powershell
scripts\setup.ps1 -SetKey -Channel baidu-ocr -Key <ak> -Secret <sk> -Verify
```

### 3. MinerU

项目地址：[https://github.com/opendatalab/MinerU](https://github.com/opendatalab/MinerU)

步骤：

1. 安装 `mineru-open-api`
2. `flash` 模式无需 token
3. `extract` 模式配置 `MINERU_TOKEN`

### 4. 本地 Ollama

```powershell
winget install Ollama.Ollama
ollama pull qwen2.5-vl:3b
```

起服务后即可走 `local` 通道：

```powershell
scripts\vlm-vision.ps1 -ImagePath <图片路径> -Prompt "描述这张图" -Channel local -Json
```

### 5. 自定义中转

任意 OpenAI 兼容服务：

```powershell
scripts\setup.ps1 -SetCustom -BaseUrl <url> -Key <key> -Model <model> -Verify
```

对应环境变量：

```text
VISION_CUSTOM_BASE_URL
VISION_CUSTOM_API_KEY
VISION_CUSTOM_MODEL
```

### 6. 查看与移除

```powershell
scripts\setup.ps1 -Status
scripts\setup.ps1 -Help
scripts\setup.ps1 -RemoveKey -Channel <glm|baidu-ocr|custom>
```

---

## Skill 位置与安装

本机安装位置（注意：本机副本没有记录公开 GitHub 仓库地址）：

```text
C:\Users\26529\.codex\skills\ds-vision-skill\
```

目录结构：

```text
ds-vision-skill/
├── SKILL.md                  # 技能定义：触发描述、路由规则、输出规范、降级与成本策略
├── README.md                 # 技能自己的说明文档
├── LICENSE                   # MIT，Copyright (c) 2026 Chenyang Wang
├── agents/
│   └── openai.yaml           # UI 元数据
├── references/
│   └── channels.md           # 通道表：模型 ID、Base URL、环境变量、注册入口、已知状态
└── scripts/
    ├── vlm-vision.ps1        # 通用视觉理解：glm / glm-thinking / custom / local
    ├── baidu-ocr.ps1         # 百度 OCR（标准/高精度）
    ├── windows-ocr.ps1       # Windows 离线 OCR
    ├── mineru-extract.ps1    # MinerU 文档解析（Markdown 输出）
    ├── preflight.ps1         # 通道可用性检查（只读）
    ├── setup.ps1             # 配置引导：Status / Help / SetKey / RemoveKey / SetCustom / Verify
    └── local-select.ps1      # llmfit 本地视觉模型选型
```

安装到新机器：

```powershell
Copy-Item -Path ".\ds-vision-skill" -Destination "$env:USERPROFILE\.codex\skills\ds-vision-skill" -Recurse
```

如果要发布到 GitHub，建议把整个 `ds-vision-skill/` 目录放进仓库，并保留 `LICENSE` 和作者版权声明。

---

## 快速开始

```powershell
# 1. 检查可用通道
scripts\setup.ps1 -Status

# 2. 简单识图（GLM-4V-Flash，免费）
scripts\vlm-vision.ps1 -ImagePath <图片路径> -Prompt "用中文描述这张图片" -Channel glm -Json

# 3. 复杂推理（图表、数学题、密集截图）
scripts\vlm-vision.ps1 -ImagePath <图片路径> -Prompt "分析这张图表的数据趋势" -Channel glm-thinking -Json

# 4. 纯文字提取（百度 OCR -> Windows 离线 OCR 兜底）
scripts\baidu-ocr.ps1 -ImagePath <图片路径> -Json
scripts\windows-ocr.ps1 -ImagePath <图片路径> -Json

# 5. PDF/文档解析（MinerU -> Markdown）
scripts\mineru-extract.ps1 -FilePath <文档路径> -Mode flash -Json
```

不依赖脚本的原生调用（OpenAI 兼容接口）：

```powershell
$key = $env:GLM_API_KEY
if (-not $key) { $key = [Environment]::GetEnvironmentVariable('GLM_API_KEY', 'User') }
if (-not $key) { throw 'GLM_API_KEY not found' }

$path = '<图片绝对路径>'
$img = [Convert]::ToBase64String([IO.File]::ReadAllBytes($path))
$body = @{
  model = 'glm-4.1v-thinking-flash'
  messages = @(@{
    role = 'user'
    content = @(
      @{ type = 'text'; text = '详细描述这张图片' },
      @{ type = 'image_url'; image_url = @{ url = "data:image/png;base64,$img" } }
    )
  })
} | ConvertTo-Json -Depth 10

$r = Invoke-RestMethod `
  -Uri 'https://open.bigmodel.cn/api/paas/v4/chat/completions' `
  -Method Post `
  -Headers @{ Authorization = "Bearer $key" } `
  -ContentType 'application/json; charset=utf-8' `
  -Body $body

$r.choices[0].message.content
```

图片处理约定：

- 图片超过 15MB 先压缩或走 MinerU
- 超过 2MB 或长边超过 2048px 自动缩放（脚本已内置）

---

## 输出规范

所有视觉工具统一输出标准 JSON：

```json
{
  "task_type": "image_reasoning | document_parsing | ocr",
  "tool_used": "实际调用的模型或工具",
  "confidence": "high | medium | low",
  "result": "视觉分析内容",
  "metadata": { "额外信息" }
}
```

字段说明：

| 字段 | 说明 |
|---|---|
| `task_type` | 任务类型：图片推理 / 文档解析 / OCR |
| `tool_used` | 实际通道，如 `glm-thinking:glm-4.1v-thinking-flash` |
| `confidence` | 置信度，OCR 乱码或结果可疑时降级为 low |
| `result` | 主模型唯一消费的内容 |
| `metadata` | 模型名、通道、图片哈希、耗时、是否命中缓存等 |

---

## 路由与降级链

```text
PDF / 论文 / 长文档 / 多页扫描
  -> MinerU（flash -> extract）

图片 + 需要理解/推理（描述、问答、图表、架构、UI、代码、数学题）
  -> glm（简单）
  -> glm-thinking（复杂）
  -> custom（中转）
  -> local（本地模型）
  -> Windows OCR 抽文字后交回主模型

图片 + 只要文字（OCR、票据、扫描件）
  -> 百度 OCR（-Accurate 高精度）
  -> Windows OCR（离线兜底）
  -> MinerU（需要版式/表格时）
```

失败规则：

- 401 / 429 / 网络错误：不要反复重试同一通道，直接降级
- OCR 质量低（乱码、缺行、置信度 low）：调用 `glm-thinking` 重新理解原图
- 全部失败：明确告诉用户失败原因，请用户描述图片

`vlm-vision.ps1` 退出码：

| 退出码 | 含义 |
|---|---|
| 0 | 成功，stdout 为内容 |
| 1 | 通用失败（文件不存在、空响应等） |
| 2 | 缺 Key / 认证失败 |
| 3 | 限流（429） |
| 4 | 网络/服务器错误 |
| 5 | 请求被拒（404/400），通常是模型 ID 失效 |

---

## 缓存与成本优化

- 缓存目录：`%USERPROFILE%\.ds-vision\cache\`
- 缓存键：图片哈希 + Prompt + 通道 + 模型
- 命中缓存直接复用，不重复调 API
- 强制重跑：`-NoCache`
- 简单任务只用免费的 `glm-4v-flash`
- 长文档优先 MinerU，不用视觉模型逐页喂图
- 本地选型结果缓存：`%USERPROFILE%\.ds-vision\local-profile.json`

---

## 隐私说明

- 云端通道（GLM、百度 OCR、MinerU、custom）会把图片发送到第三方服务器
- 介意隐私时优先 Windows OCR（不出网）或本地 Ollama 模型
- Key 保存在用户级环境变量，脚本输出只显示掩码，不打印明文

---

## 跨工具使用

这套方法不限于 Codex，Claude Code、DeepSeek 等文本模型都能用。

给其他 Agent 的最小提示语：

```text
用户发的图片在本地路径 <路径>。
先确认文件存在，再调用视觉 API 把图片转成文字/JSON，
然后基于 result 内容分析。失败时用 Windows OCR 兜底。
```

Claude Code 桌面版最容易踩的坑：

1. 子进程没有网络：先 `Test-NetConnection open.bigmodel.cn -Port 443`，不通就换带网络权限的执行方式
2. Key 没进子进程：界面里的环境变量不一定传给 Claude 启动的 shell，用用户级环境变量持久化后重开会话
3. 图片太大被 API 拒收：先压缩
4. PowerShell 5.1 中文乱码：脚本源码保持 ASCII，中文走参数传入，输出设 UTF-8
5. 同一失败通道反复重试：401/429/网络错误直接降级

---

## 常见问题排查

| 现象 | 原因 | 解决 |
|---|---|---|
| `无法连接到远程服务器` | 沙箱/子进程没有网络 | 提升权限放行网络；先验证 `Test-NetConnection` |
| 401 / 403 | Key 无效或未被子进程读取 | 检查 Process/User/Machine 三级环境变量，重新 `-Verify` |
| 404 | 模型 ID 失效 | 更新 `references/channels.md` 中的模型 ID |
| 429 | 限流 | 换通道或稍后重试，不要死循环 |
| API 拒绝 | 图片过大 | 压缩到 2MB 内 / 长边 2048px 内 |
| 中文乱码 | PowerShell 5.1 + UTF-8 | 脚本 ASCII 化，中文走参数，输出 UTF-8 |
| OCR 结果乱码 | 图片低清/复杂版式 | 用 `glm-thinking` 重看原图 |
| `mineru-open-api` not found | 未安装 | 安装 `mineru-open-api` |
| 本地端口不通 | Ollama/LM Studio 未启动 | 启动服务，确认端口 11434/1234/8080 |

---

## 验收清单

- [ ] 英文 + 中文 + 图形混合截图能识别
- [ ] 返回 JSON 包含 `task_type / tool_used / confidence / result / metadata`
- [ ] 断网/超时返回错误信息，而不是崩溃
- [ ] 同一图片同一问题命中缓存，不重复付费
- [ ] 无网络时 Windows OCR 能兜底抽文字
- [ ] PDF/表格能走 MinerU 输出 Markdown
- [ ] Key 不出现在日志或输出里

---

## 许可证

本方案使用的 `ds-vision-skill` 为 MIT 许可证，Copyright (c) 2026 Chenyang Wang。
发布时请保留 LICENSE 与版权声明。

外部服务与模型遵循各自平台的条款：

- 智谱 GLM：[open.bigmodel.cn](https://open.bigmodel.cn/)
- 百度 OCR：[百度智能云](https://console.bce.baidu.com/ai/#/ai/ocr/app/list)
- MinerU：[GitHub](https://github.com/opendatalab/MinerU)
- Ollama：[ollama.com](https://ollama.com/)
