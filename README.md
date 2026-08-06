# aigcpanel-server-dev 技能

AIGCPanel 模型服务开发技能，用于引导用户从零开发 AIGCPanel 平台的自定义 AI 模型服务（模型服务器）。

## 技能用途

当用户需要以下能力时，加载本技能：

- 🆕 **从零开发模型服务**：创建工程目录、`config.json`、`run.py`、`requirements.txt`
- ⚙️ **配置服务端**：`config.json` 的服务名称/版本/平台/入口/功能列表/`mode.type = watch` 队列模式
- 🔌 **开发任务处理**：`appPrepare` → `while watchNext()` 队列循环 → 按 `modelConfig.type` 分发 → `result` 输出
- 📚 **使用基础库**：`aigcpanelserver.py`（官方 `_aigcpanel.lib` 的轻量自包含复刻）全部接口速查
- 🧪 **测试与调试**：单次模式（`*.example.json`）、队列模式（`aigcpanel-queue/` 投递）、结果协议校验
- 🚀 **接入真实模型**：循环外加载重模型、替换模拟推理、环境依赖、发布到 AIGCPanel

## 覆盖内容

| 模块 | 说明 |
|------|------|
| 工程目录结构 | `run.py` + `aigcpanelserver.py` + `config.json` + `example-config/` + `tests/` |
| config.json 服务端配置 | name/version/platform/mode/entry/launcher/functions/settings 全字段说明 |
| 任务配置格式 | `id` / `mode` / `modelConfig.type` / `modelConfig.param` 结构 |
| 队列模式原理 | 首次返回启动配置 → 轮询 `aigcpanel-queue/*.queue.json` → 空闲超时退出 |
| 基础库接口 | `appPrepare` / `watchNext` / `result` / `resultEnd` / `resultEnv` / `urlForResult` / `localCache` / `filterText` / `end` 等 |
| 功能类型接入 | soundTts / soundClone / asr / videoGen / textToImage / imageToImage / textToVideo / imageToVideo |
| 结果输出协议 | `AigcPanelRunResult[id][base64]` 供 launcher/UI 解析 |
| 测试调试 | `tests/single.py`、`tests/queue.py`、`AIGCPANEL_SERVER_DEBUG` 调试日志 |
| 最佳实践 | 模型加载一次、结果走缓存目录、远程输入先缓存、SDK 无缝替换 |

## 文件结构

```
aigcpanel-server-dev/
├── SKILL.md           # 技能核心指令：开发流程、配置参考、接口速查、最佳实践
├── README.md          # 本文件：技能说明
└── aigcpanelserver.py # 基础库参考：官方 _aigcpanel.lib 的轻量自包含复刻版
```

## 使用方法

在会话中描述 AIGCPanel 模型服务开发需求，AI 将按 SKILL.md 中的「开发流程模板」引导：

1. 明确模型需求（功能类型、输入输出、平台设备、推理方式）
2. 创建工程骨架（参考 `aigcpanel-server-demo`，复制 `aigcpanelserver.py` 作基础库）
3. 配置 `config.json`（服务信息、`functions`、`mode.type = watch` 队列模式）
4. 开发 `run.py` 并用 `*.example.json` 单次模式验证
5. 队列联调（启动常驻服务 → 投递 `aigcpanel-queue/` 任务 → 校验结果）
6. 接入真实模型（循环外加载 → 替换模拟推理 → 建环境 → 导入 AIGCPanel 发布）

## 关联资源

- 参考工程：`aigcpanel-server-demo`（本技能文档与基础库均源自该工程）
- 基础库：`aigcpanelserver.py`（函数名/签名与官方 `_aigcpanel.lib` 一致，开发调试期使用，上线可无缝替换为官方包）
