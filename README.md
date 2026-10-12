# 银行股每日扫描（大佬刘双估值战法）

基于「大佬刘战法一」的银行股买入点扫描脚本，每日自动提示哪些银行进入上车区间。扫描结果由 `gpt-5.6-sol` 生成每日解读，并自动发布到 GitHub Pages 可视化面板。

📊 在线面板：https://lixingli0130.github.io/bank-scan/

> 本仓库 fork 自 [hyan1985/bank-scan](https://github.com/hyan1985/bank-scan)，部署与 AI 后端由 `lixingli0130` 维护。

## 估值逻辑

**核心公式**（源自文章）：
```
买入价 = min(PE估值价, 股息率估值价) + 即将派发分红
```

| 步骤 | 计算 | 说明 |
|------|------|------|
| PE 估值 | 预估自然年 EPS × 目标PE | 目标PE：优秀银行 7.67（15%年化 6.67 × 1.15），一般银行 6.67，超 10PE 不考虑 |
| 股息率估值 | 最近财年每股分红 ÷ 目标股息率 | 国有大行 5%+，优秀股份/城商行 3%+（按银行微调） |
| 取低者 | min(PE估值, 股息率估值) = 基准价 | 文章："两者取低者" |
| 加上分红 | 基准价 + 最近财年DPS | 文章："考虑到今年的分红马上要进行" |

**自然年 PE（线性预估）**：
```
预估自然年EPS = 线性回归(最近5个年度EPS, 目标自然年)
自然年PE = 现价 ÷ 预估自然年EPS
```
用最近 5 个年度 EPS 做最小二乘线性外推，得到当年自然年 EPS，再算 PE。比用静态上一年度 EPS 更贴近"全年"口径。

## 上车信号

| 信号 | 含义 |
|------|------|
| ● 射击范围 | 现价 ≤ 基准价（min双估值） |
| ○ 参考上车 | 现价 ≤ 参考上车价（基准价 + 即将分红） |
| … 等待 | 现价高于上车价，显示需再跌多少 |

## 使用

### 本地运行

```bash
cd /Users/hyan/Desktop/银行股
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

cp .env.example .env
# 编辑 .env 填入 TUSHARE_TOKEN 和 OPENAI_API_KEY

python bank_daily_scan.py
```

可选参数：
```bash
python bank_daily_scan.py --no-ai      # 跳过 OpenAI 解读
python bank_daily_scan.py --no-save    # 不保存文件
python bank_daily_scan.py --workers 4  # 并行拉取线程数
```

### 输出

- `output/bank_scan_YYYYMMDD.csv` — 全量明细
- `output/bank_scan_YYYYMMDD.xlsx` — 上车提示 Excel（含全量/上车提示/估值参数 3 个 sheet）
- `output/openai_interpret_YYYYMMDD.md` — OpenAI 每日解读

### AI 每日解读

默认通过 OpenAI Responses API 使用 `gpt-5.6-sol`，推理强度为 `medium`。可分别通过 `OPENAI_MODEL` 与 `OPENAI_REASONING_EFFORT` 覆盖；GitHub Actions 部署固定使用 `gpt-5.6-sol`，防止运行环境意外换模。

如果 OpenAI 请求失败，脚本会让本次 Action 明确失败，不会把旧日期的 AI 文本伪装成最新分析。

## 定时自动跑（GitHub Actions）

工作日（周一至周五）自动扫描，扫描结果提交回仓库并更新面板：

| 时间（北京时间） | 说明 |
|------|------|
| **21:05** | 触发（UTC `5 13`） |
| **~23:00** | 实际执行（GitHub 调度器有约 2 小时延迟） |

与其他项目错峰执行，避开 GitHub 整点高负载（`00`/`30` 整点延迟明显）。

### 部署步骤

仓库、工作流和 Pages 已部署到 `lixingli0130/bank-scan`。首次运行前只需配置 GitHub Secrets（仓库 → Settings → Secrets and variables → Actions → New repository secret）：

   - `TUSHARE_TOKEN`：你的 Tushare token（必填）
   - `OPENAI_API_KEY`：你的 OpenAI API key（必填，需有 `gpt-5.6-sol` 访问权限）

配置后可在 Actions 页面手动触发一次 **Run workflow**；之后工作日会按计划自动运行。

### 可视化面板（GitHub Pages）

- 面板模板为 `dashboard.html`（含数据占位符），扫描后由脚本注入最新数据生成 `index.html`。
- 仓库根目录的 `index.html` 即为 GitHub Pages 入口，无需额外配置构建流程。
- 打开仓库 **Settings → Pages**，Source 选择 **Deploy from a branch**，Branch 选 `main` + `/ (root)` 即可。
- Pages 部署由 GitHub 服务端完成，若出现 `deployment_queued` / 超时，多为 GitHub 服务拥堵，数据已在仓库中，服务恢复后自动部署。

### 注意

- `.env` 已被 `.gitignore` 排除，**不要提交真实 token**。GitHub Actions 只从加密 Secrets 读取密钥。
- 扫描结果自动 commit 回仓库，需给 workflow 开启 `contents: write` 权限（已在 yml 中声明）。
- 如仓库为私有，扫描结果仅自己可见；如公开，注意结果中的投资信息会公开。

## 自定义估值参数

编辑 `bank_valuation_config.csv`，按银行微调目标：

| 字段 | 含义 |
|------|------|
| `ts_code` | 6位代码（如 `600036`） |
| `name` | 银行名 |
| `type` | 国有大行/股份行/城商行/农商行 |
| `target_pe` | 目标 PE（优秀 7.67，一般 6.67） |
| `target_div_yield` | 目标股息率 %（大行 5.5，股份/城商 3~5） |
| `quality` | 质量评级（优秀/一般） |
