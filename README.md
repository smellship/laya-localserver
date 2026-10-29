# laya-localserver

本地部署 [laya](https://github.com/NandhaKishorM/laya) 并把它接到 UI 自动化上的实践仓库。

laya 是一个非自回归的 System 1 决策引擎：一次前向就能对文本给出「选项 / 打分 / 是-否」三类类型化决策
（支持 100+ 语言），权重来自 Hugging Face，可完全在本机运行，不依赖任何托管服务。本仓库把它的
本地部署、服务启停和接口调试过程整理下来，作为 UI 自动化链路里可自托管的决策后端。

## 用途

- **本地部署**：在自己的机器（CPU 或 GPU）上跑通三个 checkpoint，固定依赖版本，验证接口连通。
- **UI 测试**：配合 [Jev-browseruse-autoui](https://github.com/smellship/Jev-browseruse-autoui)
  使用——由本机 laya-serve 提供决策能力，由 ui_agent 用自然语言驱动 Playwright 执行 UI 回归。

## 目录说明

本仓库内容来自 laya 上游代码（release 0.4.6），保留完整源码以便本地修改与调试：

| 路径 | 内容 |
|---|---|
| `laya/` | Python 包本体：模型构建、路由、HTTP 服务、MCP server、CLI |
| `examples/` | 41 个递进示例，含单文件 API server `examples/server.py` |
| `sdk/`、`laya-ts/`、`laya-java/`、`laya-go/`、`laya-dotnet/` | 各语言客户端 |
| `tests/` | 测试套件，含 API 契约与可移植性检查 |
| `verify/`、`setup_laya.sh` | 本机安装脚本与端到端验证脚本 |
| `docker/`、`compose*.yaml` | 容器化部署 |

## 本地部署

### 1. 环境准备

- Python 3.10+（上游支持 3.10–3.13）
- 可选：CUDA GPU（自动识别，CPU 亦可运行，只是更慢）
- 首次运行需要联网下载权重（约 2.3 GB，三个 checkpoint）

### 2. 安装

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

pip install -e .                 # 只装核心依赖
pip install -e ".[serve]"        # 需要 HTTP 服务时加 serve extra（fastapi + uvicorn）
```

一键脚本形式（创建 venv、固定依赖、拉权重、跑一次真实前向验活）：

```bash
./setup_laya.sh                  # 幂等，重跑只补缺失部分
./setup_laya.sh --skip-models    # 跳过权重下载
```

### 3. 启动服务

两种 HTTP 形态，按需要选一个：

```bash
# A. 面向 Jev 协议的正式服务：POST /v1/systemone、/v1/systemone/batch、GET /health
LAYA_HOST=127.0.0.1 LAYA_PORT=8000 laya-serve

# B. 单文件演示服务（带网页 playground）：/predict、/predict/batch、/、/docs
python examples/server.py --host 127.0.0.1 --port 8000
```

验证服务是否真的在工作：

```bash
curl -s localhost:8000/health

curl -s localhost:8000/predict -H 'content-type: application/json' -d '{
  "state": {"subject": "Duplicate charge on invoice #4411",
            "body": "We were billed twice for March. Please refund it today."},
  "questions": {
    "department": {"type": "choice",
                   "instructions": "Which department should handle this request?",
                   "criteria": {"billing": "invoices, payments, refunds",
                                "technical": "bugs, outages, system errors"}},
    "urgent": {"type": "noul",
               "instructions": "Is this blocking the customer right now?"}
  }
}'
```

打开 `http://127.0.0.1:8000` 可以直接在 playground 里试。

### 4. 常用环境变量

`laya-serve` 全部通过环境变量配置，常用几项：

| 变量 | 含义 | 默认 |
|---|---|---|
| `LAYA_HOST` / `LAYA_PORT` | 绑定地址 / 端口 | `0.0.0.0` / `8000` |
| `LAYA_DEVICE` | 计算设备（`cpu` / `cuda`） | 自动 |
| `LAYA_MODELS` | 启动时预加载的 checkpoint 列表 | 全部 |
| `LAYA_DEFAULT_MODEL` | 无语言证据时的兜底 checkpoint | multilingual |
| `LAYA_MAX_LOADED` | 同时常驻的 checkpoint 数 | 2 |
| `LAYA_IDLE_UNLOAD_SECONDS` | 空闲多久卸载权重 | 0（关闭） |
| `LAYA_API_KEY` | 设置后要求 `Authorization: Bearer <key>` | 无 |
| `LAYA_MAX_CONCURRENT` | 并发上限，超出的返回 503 | 16 |
| `LAYA_THREADS` | CPU 推理线程数，别超过物理核数 | torch 默认 |

### 5. 容器方式

```bash
docker compose up --build          # 默认 CPU 镜像，权重缓存在 named volume
LAYA_DEVICE=cuda docker compose -f compose.cuda.yaml up --build
```

## 接到 UI 测试

UI 自动化侧在另一个仓库：[Jev-browseruse-autoui](https://github.com/smellship/Jev-browseruse-autoui)。

`ui_agent` 的思路是「一句自然语言 → 分步计划 → Jev 看 DOM 选元素 → Playwright 执行」：LLM 负责编排与
断言拆解，常态下由模型依据 DOM 快照决策「点哪个元素」，卡住时才升级给 LLM 裁决（HINT / RECOVER /
REPLAN / ABORT），所以一次回归只消耗很少的模型调用。

配合本仓库使用的典型链路：

1. 本机起 laya-serve（或 `examples/server.py`），确认 `/predict` 或 `/v1/systemone` 能返回结果；
2. 在 ui_agent 里跑目标，断言用 `--check` 独立指定，DONE 本身不算证据：

```powershell
uv run ui-agent run `
  --goal-file examples\goals\baiwang_login.txt `
  --url <被测地址> --task baiwang-login --mode debug `
  --check element_exists=行政区域
```

3. 需要回归时切 `--mode ci`（无头），断言支持轮询重试，慢半拍的页面不再白跑一轮模型；
4. 想零模型验证链路本身，ui_agent 带了一整套离线冒烟脚本（快照、点击阶梯、计划、监督、回放），
   无网络、无模型额度也能跑。

## 说明

本仓库为个人学习实践用途，用于记录本地部署过程与 UI 测试链路搭建，代码版权归 laya 上游所有
（Apache-2.0，见 [LICENSE](LICENSE)）。
