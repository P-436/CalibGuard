# 数据说明（DATA）

> 本文件定义 **CG_Bench 的数据契约**：目录结构、命名规则、前缀含义、域归属、四段划分与复现路径。
>
> **语料本身不随仓库分发。** 完整语料约 26 GB、25 万余个文件，其中包含第三方包的原始内容，授权不允许转分发。本文件的目的不是"给你数据"，而是让你**拿到（或自建）语料后，能跑出与本文一致的结果**。

---

## 1. 这份文档能解决什么

| 场景 | 是否可行 |
|---|---|
| 已有一份符合契约的语料 → 复现完全相同的四段划分与训练结果 | ✅ |
| 自建语料 → 按契约接入，复用同一套训练 / 评测 / 校准流水线 | ✅ |
| 一键下载完整语料 | ❌ 不提供（见 §6 来源与许可） |
| 只跑推理（不训练） | ✅ 直接用 `CG_GUI` 或 Releases 里的成品包，**无需任何语料** |

---

## 2. 目录契约

```
CG_Bench/
└── data/
    └── train/
        ├── malicious/    *.txt      # 恶意样本
        └── benign/       *.txt      # 良性样本
```

约定：

- **标签由所在目录决定**：`malicious/` 下即 label=1，`benign/` 下即 label=0。
- 文件名前缀的 `mal_` / `ben_` **必须与目录一致**（不一致会在清洗阶段被丢弃并记日志）。
- 文件为提取后的纯文本（`.txt`）；读取按 UTF-8，无法解码的字节以替换符处理（不中断）。
- 训练入口默认从这里读取，可用 `--data-dir` 覆盖：

```bash
python train/train.py --data-dir data/train --base-model microsoft/codebert-base
```

> `data/raw/`（原始下载物）与 `data/archive/`（历史版本划分）**不属于契约**，训练不读取它们；仓库中也不包含。

---

## 3. 命名规则

```
{mal|ben}_{prefix}_{family}_{index}.txt
```

示例：

```
mal_npm_lodash-4.17.20_0.txt
ben_py_requests_3.txt
mal_skb_openclaw-deploy-skill_17.txt
```

**家族键（family）的解析优先级**（决定"同族样本永不跨段"）：

1. 匹配四段式文件名 → 第 3 段即家族（如上面的 `lodash-4.17.20`）
2. 否则读取文件头 front-matter 的 `name:` 字段（按 `__` 截断）
3. 都没有 → 文件名自成一家族

---

## 4. 前缀 → 域

语料只有**两个域**：`code`（代码 / 包）与 `prompt`（提示词 / Skill / MCP 类）。映射表在
[`train/train.py`](train/train.py) 的 `_DOMAIN_BY_PREFIX`，**这是唯一权威口径**（评测脚本的前缀注册表若未同步，会把未登记前缀兜底归入 `other`，那不是一个真实域）。

| 前缀 | 域 | 含义 | 恶意 | 良性 |
|---|---|---|---|---|
| `py` | code | PyPI 包 | 2,480 | 2,034 |
| `npm` | code | npm 包 | 25,410 | 2,002 |
| `ide` | code | IDE 扩展代码 | 100 | — |
| `twin` | code | 良性孪生（与恶意包同族对照） | — | 504 |
| `rb` | code | RubyGems（仅良性，防误报定位） | — | 306 |
| `go` | code | Go module（仅良性） | — | 160 |
| `jv` | code | Maven sources（仅良性） | — | 173 |
| `ps` | code | PowerShell 脚本 | 12,434 | 13,633 |
| `ws` | code | Webshell | 18,392 | 33,860 |
| `ms` | prompt | MalSkillBench（同一良性原型的恶意变体） | 3,426 | 3,890 |
| `sb` | prompt | SkillBench | 150 | 1,490 |
| `atr` | prompt | ATR 良性技能 | — | 3,306 |
| `ch` | prompt | ClawHub 四扫描器共识裁决 | 206 | 2,000 |
| `sk` | prompt | DataDog ai-skills（`SKILL.md` 原文） | 146 | — |
| `mcs` | prompt | MCPShield 文本操纵筛选集 | 236 | 284 |
| `pi` | prompt | 恶意 prompt（Necent / guychuk / ahsanayub） | 59,999 | 60,000 |
| `skb` | prompt | MaliciousSkillBench | 6,436 | 1,732 |
| **code 小计** | | | **58,816** | **52,672** |
| **prompt 小计** | | | **70,599** | **72,702** |
| **合计** | | | **129,415** | **125,374** |

> 恶意 / 良性的具体条数以 `data/train/` 实际计数为准；上表为 v2.5 最终语料（清洗前）。

**为何没有 MCP 域**：v1 曾把 MCP 仓库克隆单列一域，但恶意证据在仓库元数据（F/A 级标签）而非文本层，实测召回仅 24.4%。v2 起按「只学文本层可见的恶意」原则并入两域，`mal_mcp` / `ben_mcp` 已归档。

---

## 5. 语料规模与四段划分

### 5.1 规模

| 项 | 数量 |
|---|---|
| 原始文件 | **254,789**（恶意 129,415 / 良性 125,374） |
| 去重后入模 | **250,490** |
| 剔除合计 | 4,299（raw 矛盾 2 / raw 重复 22 / 归一化矛盾 835 / 归一化重复 3,440） |
| 域分布（原始） | code 111,488（恶 58,816 / 良 52,672）；prompt 143,301（恶 70,599 / 良 72,702） |

### 5.2 四段划分

| 段 | 条数 | 用途 |
|---|---|---|
| train | 175,343 | 训练 |
| val | 25,049 | 训练中的模型选择（`load_best_model_at_end`） |
| calib | 25,049 | Platt 概率校准 |
| test | 25,049 | **仅**用于最终评估（不与模型选择共用） |

test 段构成：恶意 12,596 / 良性 12,453；按域 code 11,022（恶 5,755 / 良 5,267）、prompt 14,027（恶 6,841 / 良 7,186）。

### 5.3 划分规则

- **域感知**：`code`、`prompt` 两个域**各自**按 70/10/10/10 切分 → 每个留出段都同时含两域样本，支持分域评估。
- **group-aware**：同一家族（同包名 / 同 front-matter `name` / 同归一化指纹）**整族落在同一段**，杜绝家族级泄漏。
- **固定种子**：划分种子写死为 `42`（`_SPLIT_SEED`），与训练 `--seed` 相互独立。
- **每次训练重新划分**：`train.py` 不缓存划分，而是每次从 `data/train` 重新去重 + 划分，写出 `models/split_{train,val,calibration,test}.jsonl`。因此**只要语料与代码不变，划分就完全可复现**。
- **划分清单会被强校验**：`harness.py` 以 `results/_splits/` 作为"统一划分源"，训练后逐行比对，不一致直接中止（保证跨实验可比）。该目录**不随仓库分发**，见 §7.2 的处理方式。

### 5.4 泄露自检（v2.5 实测）

| 检查 | 结果 |
|---|---|
| 家族跨段数 | 0 ✅ |
| 六对交叉（raw-sha / 归一化 sha / 家族 × 4 段两两） | 全部 0 ✅ |
| 四组比例（code×mal/ben、prompt×mal/ben） | 均 70.0 / 10.0 / 10.0 / 10.0 ✅ |
| 留出段双类齐备（全局与分域） | PASS ✅ |
| 类别平衡（恶意 125,911 vs 良性 124,579） | 恶意略多（+0.5%），按 v2 硬规则记为 FAIL，**经确认接受、不阻断** |

完整划分报告见 `results/_splits/v25_split_report.txt`（由数据准备阶段的划分重建脚本 `data/raw/_prep/v25_split_rebuild.py` 生成；该脚本与报告均**不随仓库分发**）。

---

## 6. 来源与许可

语料由以下**公开来源**采集、清洗、按文本层核验后构造，**本仓库不转分发**：

| 来源 | 用途 | 说明 |
|---|---|---|
| [DataDog/malicious-software-packages-dataset](https://github.com/DataDog/malicious-software-packages-dataset) | 恶意代码包 | 100% 人工复核；加密 zip 密码 `infected` |
| PyPI / npm / RubyGems / Go proxy / Maven | 良性代码包 | 主流包抽样；Maven 仅取 `sources.jar` |
| HuggingFace 数据集（PowerShell / Webshell / 恶意 prompt / SkillBench） | 扩容 | 按文本层核验后入库 |
| ClawHub 扫描共识 | prompt | 仅取四扫描器一致结论，分歧样本丢弃 |
| MCPShield 扫描索引 | prompt | 仅取八类文本操纵正则命中；LLM 判官结论不采 |

**使用时请自行确认各来源的许可条款。** 部分恶意样本本身即为攻击代码（RAT 下载器、反弹 shell、凭据窃取等），仅供安全研究，请在隔离环境中处理。

---

## 7. 从零复现

> 以下命令**均在 `CG_Bench/` 目录下执行**（`train.py` / `evaluate.py` 以 `models/` 为相对工作目录）。

```bash
# 0) 环境
bash setup.sh                       # 或 setup.bat（Windows）

# 1) 准备语料：按 §2 契约放入 data/train/{malicious,benign}/
#    规模可少于 25 万条，但建议保持前缀语义与两域均有代表

# 2) 自检（确认前缀映射覆盖率 = 100%）
python - <<'PY'
import os, collections
from train.train import _DOMAIN_BY_PREFIX
c, miss = collections.Counter(), []
for lab in ("malicious", "benign"):
    for f in os.listdir(os.path.join("data", "train", lab)):
        k = "_".join(f.split("_")[:2])
        if k in _DOMAIN_BY_PREFIX:
            c[_DOMAIN_BY_PREFIX[k]] += 1
        else:
            miss.append(f)
print("域分布:", dict(c), "| 未映射:", len(miss), miss[:5])
PY

# 3) 训练（自动去重 + 划分 + 训练；split_*.jsonl 落在 models/，cwd 须为 CG_Bench/）
python train/train.py --data-dir data/train \
    --base-model microsoft/codebert-base \
    --output-dir results/codebert_ft/codebert_finetuned \
    --epochs 3 --batch-size 16 --lr 2e-5 --seed 42

# 4) 两段评测：裸测 → 校准 → 校准后评测
#    裸测用恒等参数（A=-1, B=0 不改变模型原始概率）
cat > results/codebert_ft/identity_params.json <<'JSON'
{"A": -1.0, "B": 0.0, "method": "platt", "n_samples": 0}
JSON

python train/evaluate.py --split test \
    --model-dir results/codebert_ft/codebert_finetuned \
    --params  results/codebert_ft/identity_params.json

python train/calibrate.py \
    --model-dir  results/codebert_ft/codebert_finetuned \
    --calib-file models/split_calibration.jsonl \
    --output     results/codebert_ft/calibration_params.json

python train/evaluate.py --split test \
    --model-dir results/codebert_ft/codebert_finetuned \
    --params  results/codebert_ft/calibration_params.json
```

> `evaluate.py` 的 `--split` 默认是 `val`，**最终指标必须显式传 `test`**；`calibrate.py` 默认从 `models/split_calibration.jsonl` 读校准集。

### 7.2 用编排层（harness）时的一个前提

`harness.py` 会把它认定的"统一划分源" `results/_splits/split_*.jsonl` 与 `train.py` 现场重算的划分**逐行比对，不一致直接中止**（这是保证跨实验可比性的硬约束）。该目录**不在仓库里**。

- **要复现本文的指标**：需要原始语料 + 原始 `results/_splits/`（未分发）。
- **换自己的语料**：先按上面第 3 步跑一次 `train.py`，再把生成的 `models/split_*.jsonl` 复制过去，作为你自己的划分源：

```bash
mkdir -p results/_splits
cp models/split_*.jsonl results/_splits/

bash train-bench.sh                 # 之后即可按 experiments.yaml 串行执行
python harness.py report            # 汇总报告
```

> `harness.py init` 是从 `models_baseline/v0.2.2/` 复制的，那一份也不在仓库里，所以**换语料时不要用 `init`**，按上面的 `cp` 建立即可。

---

## 8. 已知局限

1. **包级标签 = 不精确监督**：一个恶意包里并非每行代码都恶意，但整包被标为恶意（MIL 式）。模型学到的是"该包整体可疑"。
2. **只取头部 512 token 训练**：`train.py` 用 `truncation=True, max_length=512`；语料构造时已把 `setup.py` / `package.json` 等信号文件排到最前，使截断落在信息密集处。评测则走**全文滑窗**（512 窗口 / 256 步长 / 取 `max(窗口 logit)`），两者口径不同，属有意设计。
3. **语义近重复**：规范化层只能消除"仅大小写 / 空白不同"的近重复；变量重命名级语义近重复不可见，是基准的通用局限（已在划分报告如实记录）。
4. **非文本层恶意无法识别**：动态行为、仓库元数据、下架动作等证据不在文本中，此类样本已剔除而非硬标（`mal_mcp` 归档即此原因）。
5. **类别轻微失衡**：v2.5 扩容后恶意比良性多 0.5%，未做重采样（经确认接受）。
6. **恶意样本仅覆盖 Python / JS / TS / PowerShell / Webshell 等载体**：Ruby / Go / Java 目前只有良性样本（公开源中不存在可下载的恶意本体），恶意侧待后续数据源补充。

---

## 9. 相关文件

| 文件 | 说明 |
|---|---|
| [`train/train.py`](train/train.py) | 去重、分域 group-aware 划分、训练；`_DOMAIN_BY_PREFIX` 为权威域映射 |
| [`train/hposearch.py`](train/hposearch.py) | 超参搜索（网格 + val F1 选优） |
| [`train/evaluate.py`](train/evaluate.py) | 全文滑窗评测 |
| [`train/calibrate.py`](train/calibrate.py) | Platt 校准参数拟合 |
| [`experiments.yaml`](experiments.yaml) | 实验队列与选型配置 |
| [`../模型性能对比分析报告.md`](模型性能对比分析报告.md) | 四个底座的两阶段指标与分域分析 |
