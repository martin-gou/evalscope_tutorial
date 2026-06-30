# EvalScope 代码学习教程

本教程解释 EvalScope 的 native evaluation 是怎样运行的，重点对应你当前的场景：

```bash
evalscope eval \
  --model qwen-onnx \
  --api-url http://127.0.0.1:8000/v1/chat/completions \
  --api-key EMPTY \
  --eval-type openai_api \
  --datasets arc \
  --limit 5 \
  --work-dir /Users/gouguotao/nxp/Olive/models/qwen/outputs_onnx_api
```

这里 EvalScope 不直接加载 ONNX。它把 ONNX 推理服务当作一个 OpenAI-compatible HTTP 服务来调用；评测系统只关心“给定 messages，得到 completion”。

## 0. 先看完整逻辑：一条 ARC 题目如何流动

先不要看类名。把系统看成三个独立程序：

```text
程序 A：ONNX 模型服务
程序 B：EvalScope 评测器
程序 C：Dashboard（可选，只显示结果）
```

它们的关系是：

```text
Olive 输出的 ONNX 文件
       |
       v
程序 A: onnx_openai_server.py
  读取 model.onnx + model.onnx.data + tokenizer
  在 http://127.0.0.1:8000 提供 /v1/chat/completions
       ^
       | HTTP 请求 / HTTP JSON 响应
       v
程序 B: evalscope eval
  下载 ARC 数据 -> 构造题目 prompt -> 调用 API -> 判断答案 -> 写报告
       |
       v
磁盘：outputs_onnx_api/<时间戳>/...
       ^
       |
程序 C: evalscope app
  扫描 reports/*.json 并显示图表
```

### 0.1 启动时发生什么

终端 1 启动程序 A：

```bash
python /Users/gouguotao/nxp/scripts/onnx_openai_server.py \
  --model-dir /Users/gouguotao/nxp/Olive/models/qwen \
  --model-name qwen-onnx \
  --port 8000
```

这个程序一次性把 ONNX 模型和 tokenizer 加载到内存。此时它**不下载 ARC，不评测，也不写 EvalScope 报告**；它只是等待 HTTP 请求。

终端 2 启动程序 B：

```bash
evalscope eval \
  --model qwen-onnx \
  --api-url http://127.0.0.1:8000/v1/chat/completions \
  --api-key EMPTY \
  --eval-type openai_api \
  --datasets arc \
  --limit 5 \
  --work-dir /Users/gouguotao/nxp/Olive/models/qwen/outputs_onnx_api
```

此命令启动程序 B。`--model qwen-onnx` 只是 EvalScope 在 HTTP JSON 中写的模型名字；真正模型文件仍只由程序 A 持有。

### 0.2 一道选择题的逐步流动

假设 ARC 给 EvalScope 一条原始数据：

```text
question: Which object conducts electricity?
choices: A. wood  B. rubber  C. copper  D. glass
reference: C
```

以下步骤按顺序发生：

```text
1. EvalScope 下载/读取 ARC record。

2. ARC adapter 将 record 转为统一 Sample：
   Sample(input=question, choices=[...], target="C").

3. adapter 的 prompt template 把 Sample 变成一段明确要求选择答案的文本，
   再包装成内部消息：
   [ChatMessageUser(content="...question and A/B/C/D choices...")]

4. OpenAICompatibleAPI 把内部消息转为 HTTP JSON：
   POST /v1/chat/completions
   {
     "model": "qwen-onnx",
     "messages": [{"role": "user", "content": "..."}]
   }

5. ONNX wrapper 收到 JSON，将 messages 使用 Qwen 的 chat_template.jinja
   变成 token IDs，调用 ONNX Runtime GenAI 生成 token IDs，再解码为字符串。

6. wrapper 返回 OpenAI 格式 JSON：
   {
     "choices": [{"message": {"role": "assistant", "content": "C"}}],
     "usage": {...}
   }

7. EvalScope 得到 ModelOutput。ARC adapter 从 "C"（或一段含 C 的文本）
   提取最终选项 "C"，与 reference "C" 比较，得到该样本 score=1。

8. EvalScope 立即写入两份缓存：
   predictions/...jsonl 保存模型原始回答；
   reviews/...jsonl 保存提取后的答案、reference 和 score。

9. 全部样本完成后，EvalScope 计算：
   accuracy = 所有 score 之和 / 样本数，
   并写入 reports/qwen-onnx/arc.json。
```

`limit 5` 时 ARC 的 `ARC-Easy` 和 `ARC-Challenge` 都会各取最多 5 条，因此第 1 到第 8 步会分别对两个 subset 重复执行。

### 0.3 哪个组件负责什么

| 事情 | 负责者 | 不负责者 |
| --- | --- | --- |
| 加载 `model.onnx`、生成文字 | ONNX wrapper | EvalScope、Dashboard |
| 下载 ARC、构造选择题 prompt | EvalScope benchmark adapter | ONNX wrapper、Dashboard |
| 用正确答案判断对错 | EvalScope adapter/metric | ONNX wrapper |
| 写 `predictions` / `reviews` / `reports` | EvalScope | ONNX wrapper |
| 打开网页显示分数 | Dashboard | 模型服务、评分器 |

这四个责任不要混淆：模型服务只做推理；EvalScope 只做实验和评分；Dashboard 只做可视化。

### 0.4 结果目录的结构

一次成功运行后，目录应当是：

```text
outputs_onnx_api/                         # 你传给 --work-dir 的路径
└── 20260626_094333/                      # EvalScope 自动创建的一次 run
    ├── configs/
    │   └── task_config.yaml              # 本次命令最终解析出的完整配置
    ├── logs/
    │   └── eval_log.log                  # 运行日志、HTTP/数据集错误等
    ├── predictions/
    │   └── qwen-onnx/
    │       ├── arc_ARC-Easy.jsonl        # 每题的原始模型输出
    │       └── arc_ARC-Challenge.jsonl
    ├── reviews/
    │   └── qwen-onnx/
    │       ├── arc_ARC-Easy.jsonl        # 每题的答案提取和得分
    │       └── arc_ARC-Challenge.jsonl
    └── reports/
        ├── qwen-onnx/
        │   └── arc.json                  # 最终聚合指标
        └── report.html                   # 独立 HTML 报告
```

因此，调试顺序固定为：`logs` 看是否运行正常，`predictions` 看模型实际说了什么，`reviews` 看评分器如何理解回答，最后在 `reports` 看总分。

## 1. 总体架构

一次文本生成评测的路径是：

```text
CLI arguments
  -> TaskConfig
  -> LazyModel
  -> Benchmark / DataAdapter
  -> prompt messages
  -> ModelAPI.generate()
  -> HTTP POST /v1/chat/completions
  -> ModelOutput / TaskState
  -> metric / reviewer
  -> predictions/*.jsonl + reviews/*.jsonl
  -> reports/<model>/<dataset>.json + reports/report.html
  -> Dashboard scan and display
```

主要边界：

| 部件 | 责任 |
| --- | --- |
| `TaskConfig` | 保存命令行参数和整个任务配置。 |
| `ModelAPI` | 统一模型调用接口；本地 PyTorch、OpenAI API、Anthropic API 都可以实现它。 |
| `Benchmark` / `DataAdapter` | 知道如何下载一个数据集、构造 prompt、解析回答、计算指标。 |
| `DefaultEvaluator` | 编排数据加载、并发推理、缓存、评分、聚合与报告。 |
| `Report` | 序列化最终聚合结果。 |
| Dashboard | Flask API + React 前端，只读取工作目录中的报告。它不运行模型。 |

## 2. CLI 从哪里进入

入口是 [evalscope/cli/cli.py](evalscope/cli/cli.py)：

```python
subparsers = parser.add_subparsers()
EvalCMD.define_args(subparsers)
...
cmd = args.func(args)
cmd.execute()
```

`eval` 子命令的实现很薄，在 [evalscope/cli/start_eval.py](evalscope/cli/start_eval.py)：

```python
class EvalCMD(CLICommand):
    def execute(self):
        from evalscope.run import run_task
        run_task(self.args)
```

所以真正的入口是 [evalscope/run.py](evalscope/run.py) 的 `run_task()`。

## 3. 参数如何变成 TaskConfig

`run_task()` 调用 `parse_task_config()`，把 argparse 的 `Namespace`、字典或 YAML/JSON 转为 [evalscope/config.py](evalscope/config.py) 的 `TaskConfig`。

几个直接影响本次运行的字段：

```python
class TaskConfig(BaseArgument):
    model: Optional[Union[str, Model, ModelAPI]]
    model_id: Optional[str]
    datasets: List[str]
    eval_type: Optional[str]          # openai_api
    api_url: Optional[str]            # 本地 wrapper URL
    api_key: Optional[str]
    limit: Optional[Union[int, float]]
    eval_batch_size: int = 1
    work_dir: str
    use_cache: Optional[str]
```

`--work-dir` 不是单个报告文件的路径，而是**运行根目录**。默认情况下 `setup_work_directory()` 会在它下面追加时间戳：

```text
outputs_onnx_api/
  20260626_094333/
    configs/task_config.yaml
    predictions/qwen-onnx/arc_*.jsonl
    reviews/qwen-onnx/arc_*.jsonl
    reports/qwen-onnx/arc.json
    reports/report.html
    logs/eval_log.log
```

## 4. 为什么 ONNX 要通过 OpenAI API

EvalScope 的 `llm_ckpt` 后端是为 Hugging Face/PyTorch checkpoint 设计的，会寻找 `config.json`、`safetensors` 等文件。你的 Olive 输出是 ONNX Runtime GenAI 包，因此不能用：

```bash
evalscope eval --model . --dataset arc
```

作为默认 checkpoint 模型运行。

解决方式是把 ONNX Runtime GenAI 包放在一个服务后面。我们使用的 wrapper 位于：

```text
/Users/gouguotao/nxp/scripts/onnx_openai_server.py
```

其内部逻辑是：

```text
OpenAI request messages
  -> tokenizer.apply_chat_template(...)
  -> tokenizer.encode(prompt)
  -> onnxruntime_genai.Generator
  -> tokenizer.decode(tokens)
  -> OpenAI chat-completions JSON response
```

EvalScope 因此无需知道 `model.onnx`、`model.onnx.data` 或 KV cache 的细节。

## 5. `openai_api` 具体如何发请求

模型 API 注册表在 [evalscope/models/model_apis.py](evalscope/models/model_apis.py)：

```python
@register_model_api(name='openai_api')
def openai_api() -> type[ModelAPI]:
    from .openai_compatible import OpenAICompatibleAPI
    return OpenAICompatibleAPI
```

真实实现是 [evalscope/models/openai_compatible.py](evalscope/models/openai_compatible.py)。初始化时它创建 OpenAI Python SDK client：

```python
self.client = OpenAI(
    api_key=self.api_key,
    base_url=self.base_url,
)
```

`generate()` 会把 EvalScope 内部 `ChatMessage` 转成 OpenAI 格式，再调用：

```python
completion = self.client.chat.completions.create(
    messages=openai_chat_messages(input, ...),
    **completion_params,
)
```

随后把 HTTP 响应转成统一的 `ModelOutput`，并记录：

```python
PerformanceMetrics(
    latency=total_time,
    ttft=ttft,
    input_tokens=usage.input_tokens,
    output_tokens=usage.output_tokens,
)
```

这也是为什么 wrapper 应返回规范的 `choices`、`usage`、`model` 等 OpenAI 兼容字段。API 的模型名称 `qwen-onnx` 只是请求中的 `model` 字段；它不需要是 Hugging Face Hub 的模型 ID。

## 6. LazyModel：为什么不是启动时立即连接模型

在 [evalscope/api/model/lazy_model.py](evalscope/api/model/lazy_model.py)，`LazyModel` 推迟模型实例化：

```python
def __getattr__(self, name):
    model = self._ensure_loaded()
    return getattr(model, name)
```

首次真正需要推理时，`_ensure_loaded()` 调用 `get_model_with_task_config()`，依据 `eval_type=openai_api` 创建 `OpenAICompatibleAPI`。这样当全部预测都可从缓存恢复时，不必创建本地模型或 API client。

## 7. 数据集如何变成模型 prompt

每个数据集都有 benchmark metadata 和 adapter。抽象接口在 [evalscope/api/benchmark/benchmark.py](evalscope/api/benchmark/benchmark.py)：

```python
class DataAdapter(ABC):
    def load_dataset(self) -> DatasetDict: ...
    def run_inference(self, model, sample, output_dir) -> TaskState: ...
    def calculate_metrics(self, task_state) -> SampleScore: ...
    def aggregate_scores(self, sample_scores) -> List[AggScore]: ...
```

默认实现 [evalscope/api/benchmark/adapters/default_data_adapter.py](evalscope/api/benchmark/adapters/default_data_adapter.py) 负责：

1. 从 ModelScope、Hugging Face 或本地加载数据。
2. 把原始 record 转成统一 `Sample`。
3. 用 benchmark prompt template 格式化题目。
4. 根据需要加载 few-shot 示例。
5. 把字符串包装为 `ChatMessageUser`（也可加 system prompt）。

ARC 是选择题，因此使用 [evalscope/api/benchmark/adapters/multi_choice_adapter.py](evalscope/api/benchmark/adapters/multi_choice_adapter.py)。它把题目和选项格式化，模型返回后用 `parse_answers()` 提取选项字母，再与数据集 reference 比较。

这回答了“它如何知道语言模型答对了”：对于 ARC，reference 本身是正确选项，评分是规则匹配；不需要另一个 LLM。其他开放式任务可能使用 exact match、F1、ROUGE、代码执行，或者 LLM-as-a-judge，取决于各自 adapter。

注意：`--limit 5` 作用于每个 subset。ARC 有 `ARC-Easy` 和 `ARC-Challenge` 两个 subset，因此一次运行可能会评测 10 条，而不是 5 条。

## 8. DefaultEvaluator 的运行步骤

核心编排器在 [evalscope/evaluator/evaluator.py](evalscope/evaluator/evaluator.py)，类为 `DefaultEvaluator`。

其 `eval()` 明确分为四步：

```python
dataset_dict = self.benchmark.load_dataset()
context = self._collect_work_items(dataset_dict)
results_by_subset = self._run_pool(context)
agg_score_dict = self._aggregate_scores(dataset_dict, context, results_by_subset)
report = self.get_report(agg_score_dict)
```

### 8.1 缓存检查

`_collect_work_items()` 通过 `CacheManager` 将样本分成三类：

1. prediction 和 review 都已缓存：直接读取 score。
2. prediction 已缓存但还未评分：只运行 review。
3. 没有缓存：运行模型再评分。

因此重新运行同一个输出目录时，EvalScope 可能很快结束。这不是模型突然变快，而是复用了 `predictions/` 和 `reviews/` 的 JSONL。

### 8.2 并发推理和评分

`_run_pool()` 用 `run_in_threads_with_progress()` 执行 work items：

```python
run_in_threads_with_progress(
    context.work_items,
    worker,
    max_workers=self.task_config.eval_batch_size,
)
```

每个 sample 的路径是：

```text
DataAdapter.run_inference()
  -> model.generate(messages, ...)
  -> OpenAICompatibleAPI.generate()
  -> local ONNX OpenAI wrapper
  -> TaskState
  -> DataAdapter.calculate_metrics()
```

你的 wrapper 当前对 ONNX 生成使用锁，因此把 `eval_batch_size` 调得很大不一定提高吞吐，甚至可能只是让 HTTP 请求排队。先保持默认 `1`，确认正确性后再基准测试。

### 8.3 持久化

每个工作项完成后，主线程调用 `_persist_result()` 写文件：

```python
self.cache_manager.save_prediction_cache(...)
self.cache_manager.save_review_cache(...)
```

这样即使中途失败，已完成的样本仍可以在下次用 `--use-cache <timestamp-dir>` 恢复。

### 8.4 聚合与报告

`_aggregate_scores()` 把 sample score 按 subset 聚合，例如 `accuracy = correct / total`。`get_report()` 将报告写到：

```text
reports/qwen-onnx/arc.json
```

顶层 [evalscope/run.py](evalscope/run.py) 再调用 `gen_html_report_file()`，生成：

```text
reports/report.html
```

## 9. Dashboard 为什么有时显示 0 个报告

Web 服务端扫描函数在 [evalscope/utils/data_utils.py](evalscope/utils/data_utils.py)：

```python
for folder in glob.glob(os.path.join(root_path, '*')):
    reports_path = os.path.join(folder, 'reports')
    ...
```

所以 Dashboard 的输入必须是**时间戳目录的父目录**：

```text
正确： /.../outputs_onnx_api
错误： /.../outputs_onnx_api/20260626_094333
```

扫描器会自行寻找：

```text
outputs_onnx_api/<timestamp>/reports/<model>/*.json
```

Dashboard 本身的页面在 [evalscope/web/src/pages/DashboardPage.tsx](evalscope/web/src/pages/DashboardPage.tsx)，后端扫描路由在 [evalscope/service/blueprints/reports.py](evalscope/service/blueprints/reports.py)。它们不会从终端日志恢复结果，也不会扫描没有 `reports/<model>/*.json` 的目录。

## 10. 推荐阅读顺序

1. [evalscope/cli/cli.py](evalscope/cli/cli.py)：命令分发。
2. [evalscope/run.py](evalscope/run.py)：单次任务的顶层流程和输出目录。
3. [evalscope/config.py](evalscope/config.py)：`TaskConfig` 的所有可调参数。
4. [evalscope/api/model/lazy_model.py](evalscope/api/model/lazy_model.py)：模型何时创建。
5. [evalscope/models/openai_compatible.py](evalscope/models/openai_compatible.py)：OpenAI API 请求和响应适配。
6. [evalscope/api/benchmark/adapters/default_data_adapter.py](evalscope/api/benchmark/adapters/default_data_adapter.py)：数据、prompt 与推理协议。
7. [evalscope/evaluator/evaluator.py](evalscope/evaluator/evaluator.py)：缓存、并发、评分和报告。
8. [evalscope/utils/data_utils.py](evalscope/utils/data_utils.py)：Dashboard 如何读取落盘结果。

## 11. 调试一个评测时先看什么

```bash
# 评测完整配置：实际解析后的参数
cat <run-dir>/configs/task_config.yaml

# HTTP、数据集、异常日志
less <run-dir>/logs/eval_log.log

# 原始模型回答
head -n 1 <run-dir>/predictions/qwen-onnx/arc_ARC-Easy.jsonl

# 提取答案后的评分结果
head -n 1 <run-dir>/reviews/qwen-onnx/arc_ARC-Easy.jsonl

# 聚合指标和性能数据
cat <run-dir>/reports/qwen-onnx/arc.json
```

检查顺序应是：先确认 `predictions` 中模型确实返回了合理答案，再确认 `reviews` 的答案提取是否符合 prompt，最后才看 `reports` 的 accuracy。只看最后一个分数很难定位问题是在模型、服务、prompt 还是评分器。
