---
name: aigcpanel-server-dev
description: >
  AIGCPanel 模型服务开发技能。引导用户从零开发 AIGCPanel 平台的自定义 AI 模型服务（模型服务器），
  覆盖工程目录结构、config.json 服务端配置、任务配置格式、队列（watch）模式、基础库接口
  （appPrepare / watchNext / result / resultEnd / resultEnv / end）、结果输出协议
  （AigcPanelRunResult）、以及语音合成/克隆、视频合成、语音识别、文生图、图生图、文生视频、图生视频等
  常见模型功能接入流程。当用户需要开发 AIGCPanel 模型服务、创建模型服务工程、编写 run.py、
  接入 AI 模型、调试队列任务时使用。
---

# AIGCPanel 模型服务开发（aigcpanel-server-dev）

## 概述

本技能用于指导用户开发 **AIGCPanel** 平台的自定义 AI 模型服务。AIGCPanel 通过「模型服务器」的方式接入第三方 AI 模型：开发者编写一个常驻的 Python 服务进程，从 `aigcpanel-queue/` 目录轮询任务、执行模型推理、把结果以标准协议输出到 stdout，由 AIGCPanel 的 launcher / UI 解析展示。

核心开发模式只有一行：

```python
while config := aigcpanelserver.watchNext():
    ... 处理一个任务并输出结果 ...
aigcpanelserver.end()
```

关键设计：**模型只加载一次，常驻服务多个任务**（队列模式）。进程在任务之间保持存活，重模型不会为每个任务重复加载。

参考工程：`aigcpanel-server-demo`（本技能同目录下的 `aigcpanelserver.py` 即为官方 SDK `_aigcpanel.lib` 的轻量自包含复刻版，可直接作为基础库参考）。

开发流程概览：

1. 创建工程目录与必要文件（`config.json`、`run.py`、`aigcpanelserver.py`、`requirements.txt`）
2. 配置 `config.json` 服务端配置（名称、版本、平台、入口、功能列表、`mode.type = watch` 开启队列模式）
3. 编写 `run.py` 主程序：`appPrepare` → `while watchNext()` 队列循环 → 按 `modelConfig.type` 分发推理 → `result` / `resultEnd` → `end`
4. 本地用 `*.example.json` 任务配置单次运行验证
5. 开启队列模式，向 `aigcpanel-queue/` 投递任务，验证常驻消费
6. 在 AIGCPanel 中导入模型服务目录，接入真实模型发布

---

## 参考工程目录结构

以下结构是 AIGCPanel 模型服务的标准工程形态（参考 `aigcpanel-server-demo`）：

```
server-demo/
├── run.py                    # 主程序入口（appPrepare → while watchNext → end）🏠
├── aigcpanelserver.py         # 基础库：官方 _aigcpanel.lib 的轻量复刻（本技能已附带）
├── config.json                # 服务端配置（mode.type=watch 开启队列模式）
├── requirements.txt           # Python 依赖（官方 SDK 场景下为 _aigcpanel 包）
├── example-config/            # 任务配置示例（*.json 队列模式 / *.example.json 单次模式）
├── example-file/              # 模拟推理结果的示例媒体文件
├── tests/                     # 测试脚本（single.py 单次模式、queue.py 队列模式）
├── aigcpanel-queue/           # 队列目录（服务运行期投递 *.queue.json 任务）
├── _cache/                    # 远程文件下载缓存
├── _aienv/                    # conda 虚拟环境
└── config-last.json           # 最近一次任务配置（SDK 自动写入，用于排查）
```

> `aigcpanel-queue/`、`_cache/`、`config-last.json` 均为运行时自动生成/消费，发布时可以排除。

---

## 第一步：配置 config.json（服务端配置）

`config.json` 描述模型服务本身，由 AIGCPanel 读取用于导入、展示与启动服务。

```json
{
  "name": "server-demo",
  "version": "1.0.0",
  "title": "示例模型",
  "description": "模型描述",
  "deviceDescription": "设备描述",
  "platformName": "osx",
  "platformArch": "arm64",
  "serverRequire": "*",
  "mode": {
    "type": "watch",
    "watchDelay": 60
  },
  "entry": "__EasyServer__",
  "launcher": {
    "entry": "./_aienv/bin/python",
    "entryArgs": ["-u", "-m", "run", "${CONFIG}"],
    "envs": ["AAA=111", "BBB=222"]
  },
  "easyServer": {
    "entry": "./_aienv/bin/python",
    "entryArgs": ["run.py", "${CONFIG}"],
    "envs": []
  },
  "functions": ["soundTts", "soundClone", "asr", "textToImage", "imageToImage", "videoGen", "textToVideo", "imageToVideo"],
  "settings": []
}
```

| 字段 | 说明 |
|------|------|
| `name` | 服务唯一名称，AIGCPanel 内用于识别 |
| `version` | 服务版本号，语义化 `主.次.修订` |
| `title` / `description` | 展示名称与说明 |
| `platformName` | 目标平台：`win` / `osx` / `linux` |
| `platformArch` | 目标架构：如 `arm64`、`x64` |
| `serverRequire` | 依赖的 AIGCPanel 版本要求，`*` 表示任意 |
| `mode.type` | `watch` 开启队列模式（常驻轮询）；否则为单次模式 |
| `mode.watchDelay` | 空闲保活窗口（秒）。处理完一个任务后继续轮询该时长，无新任务则退出进程 |
| `entry` | 入口标记，示例使用 `__EasyServer__` |
| `launcher.entryArgs` | launcher 启动参数，`${CONFIG}` 由 AIGCPanel 替换为任务配置路径 |
| `functions` | 服务支持的功能类型列表（对应 `modelConfig.type` 取值） |
| `settings` | 用户在 AIGCPanel 界面可调的设置项（可空数组） |

---

## 第二步：编写主程序 run.py

### 基本骨架

```python
import aigcpanelserver


def run():
    config, ROOT_DIR = aigcpanelserver.appPrepare('server-demo')

    # NOTE: 重模型在循环外加载一次，所有队列任务复用。
    # 这正是队列模式存在的意义：进程常驻，模型不用每次任务重新加载。
    model = load_model()

    while config := aigcpanelserver.watchNext():
        modelConfig = config['modelConfig']

        # 每次任务上报设备信息（可选）
        aigcpanelserver.resultEnv()

        if 'param' not in modelConfig:
            modelConfig['param'] = {}

        aigcpanelserver.logInfo('TaskBegin', {
            'id': config['id'],
            'type': modelConfig.get('type'),
        })

        if modelConfig.get('type') == 'soundTts':
            ... 处理语音合成 ...
            aigcpanelserver.resultEnd()
            continue

        if modelConfig.get('type') == 'asr':
            ... 处理语音识别 ...
            aigcpanelserver.resultEnd()
            continue

        raise Exception('未知的模型类型: {}'.format(modelConfig.get('type')))

    aigcpanelserver.end()


if __name__ == '__main__':
    run()
```

### 任务配置格式

AIGCPanel 每次向服务投递一个任务配置（JSON），`run.py` 从 `config['modelConfig']` 中取参数：

```json
{
  "id": "xxx",
  "mode": "local",
  "modelConfig": {
    "type": "soundTts",
    "param": {},
    "text": "你好"
  },
  "setting": {}
}
```

- `id`：任务唯一 ID，必须存在，结果回传时携带
- `mode`：`local` 等
- `modelConfig.type`：功能类型，决定走哪个处理分支
- `modelConfig.param`：模型参数（如 seed），可为空对象
- `modelConfig.*`：按功能类型定义的其他输入字段（见下表）

### 支持的功能类型与结果格式

| 功能 | type | 输入字段（modelConfig） | 输出字段（result） |
|------|------|-------------------------|---------------------|
| 语音合成 | `soundTts` | `text` | `url` |
| 语音克隆 | `soundClone` | `text`、`promptAudio`、`promptText` | `url` |
| 语音识别 | `asr` | `audio` | `records`（`[{start, end, text}]`） |
| 视频合成 | `videoGen` | `video`、`audio` | `url`、`Duration` |
| 文生图 | `textToImage` | `text` | `url` |
| 图生图 | `imageToImage` | `image` | `url` |
| 文生视频 | `textToVideo` | `text` | `url` |
| 图生视频 | `imageToVideo` | `images[]`、`text` | `url` |

```python
# 生成文件的通用姿势：写入缓存目录 → urlForResult 处理 → result 输出
resultPath = aigcpanelserver.localCacheRandomPath('wav')
shutil.copy(soundFile, resultPath)
aigcpanelserver.result({'url': aigcpanelserver.urlForResult(resultPath)})
aigcpanelserver.resultEnd()
```

---

## 第三步：基础库 aigcpanelserver.py 接口速查

本技能附带 `aigcpanelserver.py`，是官方 `_aigcpanel.lib` 的轻量自包含复刻（仅标准库 + `requests`），函数名/签名与官方一致，**开发调试期直接使用本文件，上线时替换为官方 `_aigcpanel` 包即可，业务代码零改动**。

### 核心流程接口

| 接口 | 说明 |
|------|------|
| `appPrepare(name)` | 读取 `sys.argv[1]` 任务配置，初始化全局状态，返回 `(config, ROOT_DIR)`。必须在 `while watchNext()` 之前调用恰好一次 |
| `watchNext()` | 返回下一个任务配置。首次调用返回启动配置；之后若 `config.json` 中 `mode.type == 'watch'`，轮询 `aigcpanel-queue/*.queue.json`（空闲窗口 = `watchDelay` 秒）；无任务或超时返回 `None` 结束循环 |
| `end(killChildren=False)` | 清理临时文件并退出进程（exit 0） |

### 结果输出接口（launcher / UI 解析）

| 接口 | 说明 |
|------|------|
| `result(data)` | 输出任务结果。打印 `Result[id][json]` 与 `AigcPanelRunResult[id][base64]` 两行，后者是 launcher/UI 的解析协议 |
| `resultValue(key, value)` | 输出单键结果，等价 `result({key: value})` |
| `resultEnd()` | 清理临时文件并输出 `{'End': True}`，表示本任务结束 |
| `resultEnv()` | 输出设备信息（Device / DeviceName / DeviceMemorySize / CudaVersion） |
| `urlForResult(url)` | 准备结果文件路径：launcher API 模式下复制到 `launcher-data/`；schedule 模式加 `urlForResult://` 前缀 |

### 工具接口

| 接口 | 说明 |
|------|------|
| `localCache(pathOrUrl)` | 远程 URL 下载到 `_cache/_file/`（按 md5 缓存）返回本地路径；本地路径原样返回 |
| `localCacheRandomPath(ext)` | 在 `_cache/` 生成随机临时文件名，用于写推理结果 |
| `downloadFileDirect(url, path)` | 直接流式下载文件 |
| `filterText(text, filters=['emoji', 'invisible'])` | 过滤 emoji / 不可见字符，返回净化后的文本 |
| `logInfo(msg, *args)` / `logDebug` / `log` | 统一格式日志 `[I] 2026-08-06 10:00:00 - msg - args`（`flush=True` 保证输出实时） |
| `fileCleanAdd(path)` / `fileCleanRun()` | 登记/执行临时文件清理 |
| `getDevice()` / `getDeviceName()` / `getDeviceMemorySize()` / `getCudaVersion()` | 设备信息探测（torch 可选，无 torch 时优雅降级为 CPU） |
| `getServerConfig(key, default)` | 读取 `config.json` 服务端配置 |
| `getEnv` / `getEnvBool` / `setEnv` | 环境变量读写（如 `AIGCPANEL_SERVER_DEBUG=1` 开启调试日志） |

---

## 第四步：队列模式工作原理

```
run.py
  │  appPrepare()            读取启动配置，初始化环境
  ▼
while config := watchNext():   ─┐
  │  处理一个任务并输出结果      │ 首轮返回启动配置
  │                            │ 之后持续轮询 aigcpanel-queue/
  ▼                            │ 中的 *.queue.json 新任务
（无新任务到达超过 watchDelay 秒）
  ▼
end()                         退出进程
```

- 首个 `watchNext()` 返回启动时传入的配置（`sys.argv[1]`）
- 之后每次调用轮询 `aigcpanel-queue/` 目录，按文件名顺序消费 `*.queue.json`，处理完立即删除该队列文件
- `config.json` 中 `mode.watchDelay`（秒）控制空闲保活窗口，超时无新任务则返回 `None` 退出
- 启动配置命名为 `*.example.json` 时进入**单次模式**：处理一个任务后立即退出（测试用）

本地验证队列模式：

```shell
# 终端 1 - 启动常驻服务
python run.py example-config/soundTts.json

# 终端 2 - 投递任务（可任意多次）
python -c "import json,os; json.dump(json.load(open('example-config/textToImage.json')), open('aigcpanel-queue/t1.queue.json','w'), ensure_ascii=False)"
python -c "import json,os; json.dump(json.load(open('example-config/asr.json')), open('aigcpanel-queue/t2.queue.json','w'), ensure_ascii=False)"
```

服务每处理完一个任务输出一行 `AigcPanelRunResult[id][base64]`，队列文件随即被删除。

---

## 开发流程模板（给用户的引导步骤）

当用户提出 AIGCPanel 模型服务开发需求时，按以下顺序引导：

### 1. 明确模型需求

询问/确认：
- 模型功能类型（语音合成/克隆、ASR、文生图、图生图、文生视频、图生视频、视频合成等）
- 输入参数与输出结果格式
- 目标平台（win / osx / linux）与设备（CUDA / MPS / CPU）
- 模型推理方式（本地加载 / 调用远端 API）与依赖

### 2. 创建工程骨架

参考 `aigcpanel-server-demo` 结构，复制 `aigcpanelserver.py` 作为基础库，创建 `config.json`、`run.py`、`requirements.txt`，补齐各功能类型的 `example-config/*.json`。

### 3. 配置 config.json

- 填写 `name`、`version`、`title`、`platformName`、`platformArch`
- `functions` 声明支持的功能类型
- `mode.type = "watch"` + `watchDelay` 开启队列常驻模式
- 按平台调整 `launcher` / `easyServer` 的入口与 `envs`

### 4. 开发 run.py 并单次验证

- 按「基本骨架」实现 `while watchNext()` 循环，每个功能类型一个分支
- 使用 `*.example.json` 启动配置跑单次模式，确认 `AigcPanelRunResult` 输出与结果文件

### 5. 队列联调

- 用 `config.json`（watch 模式）启动常驻服务，向 `aigcpanel-queue/` 投递多个任务
- 验证：按文件名顺序消费、结果正确、队列文件删除、空闲超时退出

### 6. 接入真实模型与发布

- 在 `while` 循环外加载重模型，替换各分支中的模拟推理代码
- 确认依赖写入 `requirements.txt`，在目标平台创建 `_aienv` 虚拟环境
- 将工程目录导入 AIGCPanel 进行端到端验证后发布

---

## 测试与调试

```shell
# 非队列（单次）模式测试
python tests/single.py

# 队列模式测试：子进程启动服务 -> 投递任务 -> 校验所有结果
python tests/queue.py
```

测试断言要点：
- 退出码为 0（单次模式处理完即退出）
- 每个任务 ID 都有一行 `AigcPanelRunResult[id][base64]`
- 结果中 `url` 指向的文件真实存在
- 队列文件消费后已被删除

调试技巧：
- 设置 `AIGCPANEL_SERVER_DEBUG=1` 开启 debug 日志
- 每次任务的配置文件会写入 `config-last.json`，用于排查入参
- stdout 以 `-u` 无缓冲运行，保证结果行实时输出

---

## 最佳实践

- **模型加载一次**：重模型、tokenizer 等在 `while watchNext()` 循环外加载，队列模式专为此设计
- **结果文件走缓存目录**：推理结果写入 `localCacheRandomPath(ext)` 并用 `urlForResult()` 处理后再 `result()` 输出
- **远程输入先缓存**：所有 URL 形式的输入（音频/视频/图片）先用 `localCache()` 下载为本地文件
- **文本净化**：TTS 等文本输入先过 `filterText()`，避免 emoji / 不可见字符
- **每个任务以 `resultEnd()` 收尾**：确保临时文件清理与 `{'End': True}` 标记
- **未知类型必须抛错**：`raise Exception('未知的模型类型: ...')` 让异常可见，而不是静默跳过
- **标准输出即协议**：日志与结果都走 stdout，用 `logInfo` / `result` 保证格式统一、行可解析
- **SDK 无缝替换**：开发期用本技能附带的 `aigcpanelserver.py`，上线替换为官方 `_aigcpanel` 包，业务代码零改动

---

## 参考资料

- 参考工程：`aigcpanel-server-demo`（aigcpanelserver.py 同源）
- 基础库：本技能目录下的 `aigcpanelserver.py`（官方 `_aigcpanel.lib` 的轻量复刻，接口签名一致）
- 官方 SDK：完整版为 `_aigcpanel` Python 包，接口同本文件
