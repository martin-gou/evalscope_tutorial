# EvalScope 源码学习教程

这个文档重点解释 EvalScope 的 native evaluation 是怎么运作的，尤其对应你现在的场景：

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

先记住一句话：

```text
EvalScope 本身不是模型推理引擎。
EvalScope 是评测编排器：读数据 -> 造 prompt -> 调模型 -> 抽答案 -> 算分 -> 写报告。
```

你的 ONNX 模型不被 EvalScope 直接加载。它通过一个 OpenAI-compatible HTTP wrapper 暴露成 `/v1/chat/completions`，EvalScope 只把它当作一个 API 模型来调用。

## 1. 三个程序

你现在实际跑的是三个东西：

```text
程序 A：ONNX 模型服务
程序 B：EvalScope evaluation runner
程序 C：EvalScope Dashboard
```

关系如下：

```text
Olive 输出的 ONNX 模型目录
  model.onnx
  model.onnx.data
  tokenizer.json
  chat_template.jinja
        |
        v
程序 A：scripts/onnx_openai_server.py
  加载 ONNX Runtime GenAI 模型
  提供 HTTP endpoint:
  http://127.0.0.1:8000/v1/chat/completions
        ^
        |
        | OpenAI 格式 HTTP JSON
        |
        v
程序 B：evalscope eval
  下载/读取 benchmark 数据集
  构造 prompt
  调用程序 A
  解析模型输出
  对比标准答案
  写 predictions / reviews / reports
        |
        v
outputs_onnx_api/<timestamp>/
        ^
        |
程序 C：evalscope app
  扫描 reports/*.json
  显示图表
```

不要把这三件事混在一起：

| 组件 | 负责什么 | 不负责什么 |
| --- | --- | --- |
| ONNX wrapper | 加载 `model.onnx`，生成文字，返回 OpenAI JSON | 不知道 ARC，不知道正确答案，不写 EvalScope 报告 |
| EvalScope runner | 读数据集，发请求，抽答案，算分，写报告 | 不直接执行 ONNX graph |
| Dashboard | 读报告并可视化 | 不跑模型，不重新评分 |

## 2. 一条 ARC 样本的完整流动

假设 ARC 数据集里有一条原始记录：

```json
{
  "question": "Which material conducts electricity?",
  "choices": {
    "text": ["wood", "rubber", "copper", "glass"]
  },
  "answerKey": "C"
}
```

EvalScope 会这样处理：

```text
1. DataLoader 读取原始 record。

2. ARCAdapter.record_to_sample(record) 把 record 转成 Sample：
   Sample(
     input="Which material conducts electricity?",
     choices=["wood", "rubber", "copper", "glass"],
     target="C"
   )

3. MultiChoiceAdapter.format_prompt_template(sample) 构造选择题 prompt：
   Answer the following multiple choice question...
   A) wood
   B) rubber
   C) copper
   D) glass

4. DefaultDataAdapter.process_sample_str_input() 把 prompt 包装成 message：
   [ChatMessageUser(content="...")]

5. DefaultEvaluator._predict_sample() 调用：
   benchmark.run_inference(model, sample, output_dir)

6. DefaultDataAdapter.run_inference() 调用：
   model.generate(input=sample.input, tools=sample.tools)

7. LazyModel 第一次被访问时加载真正模型 API：
   get_model_with_task_config(task_config)
   eval_type=openai_api -> OpenAICompatibleAPI

8. OpenAICompatibleAPI.generate() 把内部 ChatMessage 转成 OpenAI JSON：
   POST http://127.0.0.1:8000/v1/chat/completions
   {
     "model": "qwen-onnx",
     "messages": [{"role": "user", "content": "..."}]
   }

9. ONNX wrapper 收到请求：
   messages -> chat_template.jinja -> token ids -> ONNX Runtime GenAI -> text

10. ONNX wrapper 返回：
    {
      "choices": [
        {"message": {"role": "assistant", "content": "ANSWER: C"}}
      ]
    }

11. OpenAICompatibleAPI 把返回 JSON 转成 ModelOutput。

12. DefaultDataAdapter._on_inference_end() 把 Sample + ModelOutput 包成 TaskState。

13. DefaultEvaluator._review_task_state() 调用：
    benchmark.calculate_metrics(task_state)

14. MultiChoiceAdapter.extract_answer() 从模型输出中解析出 "C"。

15. DefaultDataAdapter.match_score() 调用 acc metric：
    prediction="C", reference="C" -> score=1

16. CacheManager 写：
    predictions/qwen-onnx/arc_ARC-Easy.jsonl
    reviews/qwen-onnx/arc_ARC-Easy.jsonl

17. 所有样本完成后：
    aggregate_scores() 计算平均 acc。

18. generate_report() 写：
    reports/qwen-onnx/arc.json
    reports/report.html
```

这里最重要的是 `Sample -> ChatMessage -> ModelOutput -> TaskState -> SampleScore -> Report` 这条数据流。

## 3. 真实 call stack

下面是你运行 `evalscope eval ...` 之后的主调用链。文件路径都是相对 repo 根目录。

```text
evalscope/cli/cli.py
  main CLI parser
  |
  v
evalscope/cli/start_eval.py
  EvalCMD.execute()
  |
  v
evalscope/run.py
  run_task()
  parse_task_config()
  run_single_task()
  setup_work_directory()
  evaluate_model()
  |
  v
evalscope/api/model/lazy_model.py
  LazyModel(task_config)
  |
  v
evalscope/api/registry.py
  get_benchmark(dataset_name, task_config)
  create_evaluator(...)
  |
  v
evalscope/evaluator/evaluator.py
  DefaultEvaluator.eval()
  |
  +--> benchmark.load_dataset()
  |      |
  |      v
  |    evalscope/api/benchmark/adapters/default_data_adapter.py
  |      DefaultDataAdapter.load_dataset()
  |      load()
  |      load_from_remote() / load_from_disk()
  |      load_subsets()
  |      load_subset()
  |      record_to_sample()
  |      _post_process_samples()
  |
  +--> _collect_work_items()
  |
  +--> _run_pool()
  |      |
  |      v
  |    _process_work_item()
  |      |
  |      +--> _predict_sample()
  |      |      |
  |      |      v
  |      |    benchmark.run_inference()
  |      |      DefaultDataAdapter.run_inference()
  |      |      _on_inference()
  |      |      model.generate()
  |      |        |
  |      |        v
  |      |      LazyModel._ensure_loaded()
  |      |      get_model_with_task_config()
  |      |      OpenAICompatibleAPI.generate()
  |      |      HTTP POST /v1/chat/completions
  |      |
  |      +--> _review_task_state()
  |             |
  |             v
  |           benchmark.calculate_metrics()
  |           filter_prediction()
  |           extract_answer()
  |           match_score()
  |
  +--> _aggregate_scores()
  |      benchmark.aggregate_scores()
  |
  +--> get_report()
         benchmark.generate_report()
         Report.to_json()
```

如果你调试时不知道在哪里看，按这个顺序看源码：

1. `evalscope/run.py`
2. `evalscope/evaluator/evaluator.py`
3. `evalscope/api/benchmark/adapters/default_data_adapter.py`
4. `evalscope/api/benchmark/adapters/multi_choice_adapter.py`
5. `evalscope/benchmarks/arc/arc_adapter.py`
6. `evalscope/models/openai_compatible.py`
7. `evalscope/api/model/lazy_model.py`
8. `evalscope/api/registry.py`

## 4. 核心对象

### 4.1 TaskConfig

源码：`evalscope/config.py`

`TaskConfig` 是命令行参数解析后的任务配置。你的命令大概会变成：

```python
TaskConfig(
    model="qwen-onnx",
    model_id="qwen-onnx",
    eval_type="openai_api",
    api_url="http://127.0.0.1:8000/v1/chat/completions",
    api_key="EMPTY",
    datasets=["arc"],
    limit=5,
    work_dir="/Users/gouguotao/nxp/Olive/models/qwen/outputs_onnx_api/<timestamp>",
)
```

注意：

```text
--work-dir 不是最终报告文件路径。
--work-dir 是本次评测输出根目录。
默认会在它下面再创建一个 timestamp 子目录。
```

### 4.2 BenchmarkMeta

源码：`evalscope/api/benchmark/meta.py`

`BenchmarkMeta` 是一个 benchmark 的静态描述：

```python
BenchmarkMeta(
    name='arc',
    pretty_name='ARC',
    dataset_id='allenai/ai2_arc',
    subset_list=['ARC-Easy', 'ARC-Challenge'],
    metric_list=['acc'],
    train_split='train',
    eval_split='test',
    prompt_template=MultipleChoiceTemplate.SINGLE_ANSWER,
)
```

它回答这些问题：

```text
这个 evaluation 叫什么？
数据集在哪里？
有哪些 subset？
用哪个 split？
用什么 metric？
默认 prompt template 是什么？
```

### 4.3 DataAdapter

源码：

```text
evalscope/api/benchmark/benchmark.py
evalscope/api/benchmark/adapters/default_data_adapter.py
evalscope/api/benchmark/adapters/multi_choice_adapter.py
```

`DataAdapter` 是新增 evaluation 最重要的地方。

它负责：

```text
原始 record -> Sample
Sample -> prompt messages
模型输出 -> 提取最终答案
最终答案 + reference -> 单样本分数
单样本分数列表 -> 聚合指标
聚合指标 -> Report
```

常用父类：

| 父类 | 适合什么任务 |
| --- | --- |
| `MultiChoiceAdapter` | A/B/C/D 选择题，比如 ARC、MMLU、CommonsenseQA |
| `DefaultDataAdapter` | 普通问答、代码、数学、分类、需要自定义评分的任务 |
| `VisionLanguageAdapter` | 图文输入 |
| `NERAdapter` | 命名实体识别 |
| `AgentAdapter` | 多轮 agent/tool 任务 |

### 4.4 Sample

源码：`evalscope/api/dataset/dataset.py`

`Sample` 是 EvalScope 内部统一的样本格式。一个 benchmark 的原始字段可能叫 `question`、`query`、`prompt`、`answerKey`、`label`，但进入评测流程后都会变成 `Sample`。

选择题典型结构：

```python
Sample(
    input="Which material conducts electricity?",
    choices=["wood", "rubber", "copper", "glass"],
    target="C",
    metadata={"id": "..."},
)
```

普通问答典型结构：

```python
Sample(
    input="What is the capital of France?",
    target="Paris",
    metadata={}
)
```

如果你已经自己构造好了 chat messages，也可以：

```python
Sample(
    input=[ChatMessageUser(content="...")],
    target="...",
)
```

### 4.5 LazyModel

源码：`evalscope/api/model/lazy_model.py`

`LazyModel` 的作用是延迟加载模型。创建 evaluator 时不会马上加载模型；第一次真正调用 `model.generate()` 时才会创建实际 model object。

为什么这样设计：

```text
如果 predictions/reviews 已经有缓存，可能完全不需要重新加载模型。
LazyModel 可以避免无意义的模型初始化开销。
```

在你的场景里，第一次 `model.generate()` 会触发：

```text
LazyModel._ensure_loaded()
  -> get_model_with_task_config(task_config)
  -> eval_type=openai_api
  -> OpenAICompatibleAPI(...)
```

### 4.6 ModelAPI / OpenAICompatibleAPI

源码：

```text
evalscope/api/model/model.py
evalscope/models/openai_compatible.py
```

`ModelAPI` 是统一模型接口。EvalScope 不希望 benchmark 关心底层是 PyTorch、OpenAI API、Anthropic API 还是别的服务，所以统一成：

```python
model.generate(input: List[ChatMessage], tools=...)
```

你的 ONNX 模型走的是：

```text
OpenAICompatibleAPI.generate()
  -> openai_chat_messages()
  -> self.client.chat.completions.create(...)
  -> model_output_from_openai(...)
```

这里的 `self.client` 是 OpenAI Python SDK，但 `base_url` 指向你的本地服务：

```text
http://127.0.0.1:8000/v1
```

所以实际请求会发给：

```text
http://127.0.0.1:8000/v1/chat/completions
```

### 4.7 TaskState

源码：`evalscope/api/evaluator/state.py`

`TaskState` 是一个样本推理完成后的完整状态：

```text
model 名字
sample 原始样本
messages 对话历史
output 模型输出
target 标准答案
metadata 样本附加信息
completed 是否完成
```

评分阶段不直接拿原始 record，而是拿 `TaskState`。

### 4.8 SampleScore / AggScore / Report

源码：

```text
evalscope/api/metric/scorer.py
evalscope/report/report.py
```

它们对应三个粒度：

```text
SampleScore: 一道题的分数
AggScore: 一个 subset 的聚合分数，比如 ARC-Easy acc
Report: 一个 benchmark 的完整报告，比如 qwen-onnx on arc
```

## 5. 注册机制

EvalScope 大量使用 registry。源码在 `evalscope/api/registry.py`。

### 5.1 benchmark 注册

ARC 是这样注册的：

```python
@register_benchmark(
    BenchmarkMeta(
        name='arc',
        dataset_id='allenai/ai2_arc',
        subset_list=['ARC-Easy', 'ARC-Challenge'],
        metric_list=['acc'],
        eval_split='test',
        prompt_template=MultipleChoiceTemplate.SINGLE_ANSWER,
    )
)
class ARCAdapter(MultiChoiceAdapter):
    ...
```

注册之后，命令行里才能写：

```bash
--datasets arc
```

`get_benchmark('arc', task_config)` 会：

```text
1. 从 BENCHMARK_REGISTRY 找到 name='arc' 的 BenchmarkMeta
2. deepcopy 一份 metadata
3. 用 dataset_args 覆盖部分参数
4. 拿 metadata.data_adapter，也就是 ARCAdapter
5. 返回 ARCAdapter(benchmark_meta=metadata, task_config=task_config)
```

### 5.2 evaluator 注册

默认 evaluator 是：

```python
@register_evaluator('default')
class DefaultEvaluator(Evaluator):
    ...
```

`create_evaluator()` 逻辑：

```text
如果有 benchmark 专属 evaluator，就用专属的。
否则用 default evaluator。
```

大多数 benchmark 不需要写自己的 evaluator，只写 adapter 就够。

### 5.3 metric 注册

metric 也走 registry。`metric_list=['acc']` 会让 `DefaultDataAdapter.match_score()` 调：

```python
metric_scorer = get_metric('acc')
metric_func = metric_scorer()
metric_score = metric_func(prediction=filtered_prediction, reference=reference)
```

所以新增 evaluation 时通常优先复用已有 metric，比如：

```text
acc
exact_match
f1
rouge
```

只有已有 metric 不够时，才新增 metric。

## 6. `DefaultEvaluator` 四个阶段

源码：`evalscope/evaluator/evaluator.py`

`DefaultEvaluator.eval()` 是核心执行器。它分四个阶段：

```text
Phase 1: load dataset
Phase 2: collect work items
Phase 3: run pool: predict + review
Phase 4: aggregate + report
```

### 6.1 Phase 1: 加载数据

```python
dataset_dict = {
    k: v for k, v in self.benchmark.load_dataset().items() if len(v) > 0
}
```

对于 ARC：

```text
benchmark = ARCAdapter
benchmark.load_dataset()
  -> DefaultDataAdapter.load_dataset()
  -> load_from_remote()
  -> load_subsets()
  -> load_subset("ARC-Easy")
  -> load_subset("ARC-Challenge")
```

得到：

```python
{
    "ARC-Easy": Dataset([...]),
    "ARC-Challenge": Dataset([...]),
}
```

### 6.2 Phase 2: 收集 work items

```python
context = self._collect_work_items(dataset_dict)
```

这一步决定每个样本要不要重新跑。

每个样本有三种状态：

```text
1. prediction + review 都有缓存
   -> 跳过，直接用旧分数

2. prediction 有缓存，但 review 没有
   -> 不调用模型，只重新评分

3. 没有 prediction
   -> 调用模型，然后评分
```

这就是为什么 EvalScope 有 `predictions` 和 `reviews` 两层缓存。

### 6.3 Phase 3: 并发执行样本

```python
results_by_subset = self._run_pool(context)
```

内部调用：

```text
run_in_threads_with_progress(
    context.work_items,
    worker,
    max_workers=task_config.eval_batch_size,
)
```

每个 worker 做：

```text
_process_work_item()
  -> _predict_sample()
  -> _review_task_state()
```

`eval_batch_size` 在这里控制并发 worker 数。对 API 模型来说，它影响同时发多少个 HTTP 请求。

### 6.4 Phase 4: 聚合和报告

```python
agg_score_dict = self._aggregate_scores(...)
report = self.get_report(agg_score_dict)
```

最终写：

```text
reports/<model_name>/<benchmark_name>.json
reports/report.html
```

Dashboard 只会读这些报告文件。

## 7. `DefaultDataAdapter` 的 hooks

新增 evaluation 时，你主要关心这些方法。

### 7.1 `record_to_sample()`

必须实现。它把数据集的一条原始 record 转成 `Sample`。

选择题例子：`evalscope/benchmarks/arc/arc_adapter.py`

```python
def record_to_sample(self, record) -> Sample:
    choice_texts = record['choices']['text']
    answer_key = record['answerKey']

    return Sample(
        input=record['question'],
        choices=choice_texts,
        target=answer_key,
        metadata={'id': record.get('id', '')},
    )
```

### 7.2 `format_prompt_template()`

默认逻辑：

```python
return self.prompt_template.format(question=sample.input)
```

如果继承 `MultiChoiceAdapter`，它会自动把 choices 也填进去：

```text
question + A/B/C/D choices + output format instruction
```

所以选择题通常不需要自己写 `format_prompt_template()`。

### 7.3 `extract_answer()`

用于从模型长回答中抽最终答案。

选择题父类 `MultiChoiceAdapter` 已经实现：

```text
"ANSWER: C" -> "C"
"The answer is C because..." -> "C"
```

如果你的任务输出是 JSON、代码、数学答案、多个标签，你通常要重写它。

例子：

```python
def extract_answer(self, prediction: str, task_state: TaskState) -> str:
    # 从模型输出里解析最终答案
    return prediction.strip().splitlines()[-1]
```

### 7.4 `match_score()`

默认逻辑是：

```text
filtered_prediction + reference -> metric_list 里的 metric
```

如果是简单 acc / exact_match，不用改。

如果你的评分需要执行代码、比较 JSON、调用 sandbox、调用 LLM judge，就重写它。

HumanEval 就重写了 `match_score()`，因为它要执行代码测试。

### 7.5 `aggregate_scores()`

默认用 metadata 里的 `aggregation` 聚合，比如 mean。

多数任务不用改。

如果你需要 `pass@k`、分组统计、多个维度统计，可以重写或换 aggregation。

### 7.6 `generate_report()`

多数任务不用改。默认会生成 EvalScope 标准 report。

只有当你想写非常特殊的 report 结构时才改。

## 8. OpenAI API 后端怎样接 ONNX

你的命令：

```bash
evalscope eval \
  --model qwen-onnx \
  --api-url http://127.0.0.1:8000/v1/chat/completions \
  --api-key EMPTY \
  --eval-type openai_api \
  --datasets arc
```

关键点：

```text
--eval-type openai_api
  告诉 EvalScope：不要加载本地 PyTorch checkpoint，使用 OpenAI-compatible API。

--model qwen-onnx
  只是 API 请求里的 model 字段，也是报告里的 model name。

--api-url http://127.0.0.1:8000/v1/chat/completions
  指向你的本地 ONNX wrapper。

--api-key EMPTY
  OpenAI Python SDK 要求有 key；本地 wrapper 通常不校验，所以填 EMPTY。
```

EvalScope 发送的请求大概是：

```json
{
  "model": "qwen-onnx",
  "messages": [
    {
      "role": "user",
      "content": "Answer the following multiple choice question..."
    }
  ],
  "temperature": 0,
  "max_tokens": 2048
}
```

ONNX wrapper 必须返回 OpenAI-compatible response：

```json
{
  "id": "chatcmpl-local",
  "object": "chat.completion",
  "created": 1234567890,
  "model": "qwen-onnx",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "ANSWER: C"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 100,
    "completion_tokens": 5,
    "total_tokens": 105
  }
}
```

只要 wrapper 满足这个协议，EvalScope 不关心底层是 ONNX、vLLM、llama.cpp、Ollama 还是云端模型。

## 9. 输出目录怎么看

一次成功运行后：

```text
outputs_onnx_api/
└── 20260626_094333/
    ├── configs/
    │   └── task_config.yaml
    ├── logs/
    │   └── eval_log.log
    ├── predictions/
    │   └── qwen-onnx/
    │       ├── arc_ARC-Easy.jsonl
    │       └── arc_ARC-Challenge.jsonl
    ├── reviews/
    │   └── qwen-onnx/
    │       ├── arc_ARC-Easy.jsonl
    │       └── arc_ARC-Challenge.jsonl
    └── reports/
        ├── qwen-onnx/
        │   └── arc.json
        └── report.html
```

调试顺序：

1. `logs/eval_log.log`
   看数据集下载、API 调用、异常堆栈。

2. `predictions/...jsonl`
   看模型原始输出。比如模型是不是没有按 `ANSWER: C` 输出。

3. `reviews/...jsonl`
   看 EvalScope 抽出来的答案是什么，以及 reference 是什么。

4. `reports/...json`
   看最终聚合分数。

Dashboard 扫描路径要填 `--work-dir` 的父目录：

```text
/Users/gouguotao/nxp/Olive/models/qwen/outputs_onnx_api
```

不要填 timestamp 子目录：

```text
/Users/gouguotao/nxp/Olive/models/qwen/outputs_onnx_api/20260626_094333
```

原因是 Dashboard 的 scanner 会在你给的 root 下查找：

```text
root/*
  reports/
    <model>/
      <dataset>.json
```

## 10. 新增一个 evaluation：你应该改哪里

新增 evaluation 时，先判断它是哪类任务。

### 10.1 选择题任务

比如你的数据是：

```json
{
  "question": "2 + 2 = ?",
  "options": ["1", "2", "3", "4"],
  "answer": "D"
}
```

最小实现：

```python
from evalscope.api.benchmark import BenchmarkMeta, MultiChoiceAdapter
from evalscope.api.dataset import Sample
from evalscope.api.registry import register_benchmark
from evalscope.constants import Tags
from evalscope.utils.multi_choices import MultipleChoiceTemplate


@register_benchmark(
    BenchmarkMeta(
        name='my_mcq',
        pretty_name='My MCQ',
        tags=[Tags.MULTIPLE_CHOICE],
        dataset_id='/path/to/my_dataset',
        subset_list=['default'],
        metric_list=['acc'],
        eval_split='test',
        prompt_template=MultipleChoiceTemplate.SINGLE_ANSWER,
    )
)
class MyMCQAdapter(MultiChoiceAdapter):

    def record_to_sample(self, record) -> Sample:
        return Sample(
            input=record['question'],
            choices=record['options'],
            target=record['answer'],
            metadata={'id': record.get('id', '')},
        )
```

放置位置可以类似：

```text
evalscope/evalscope/benchmarks/my_mcq/my_mcq_adapter.py
```

还要确保这个模块会被 import。EvalScope 的 benchmarks 通常通过 package import 注册；如果你新增目录，要检查 `evalscope/evalscope/benchmarks/__init__.py` 或相关自动导入逻辑是否覆盖到你的文件。

运行：

```bash
evalscope eval \
  --model qwen-onnx \
  --api-url http://127.0.0.1:8000/v1/chat/completions \
  --api-key EMPTY \
  --eval-type openai_api \
  --datasets my_mcq \
  --limit 10 \
  --work-dir outputs_my_mcq
```

### 10.2 普通问答任务

比如数据是：

```json
{
  "query": "What is the capital of France?",
  "answer": "Paris"
}
```

可以继承 `DefaultDataAdapter`：

```python
from evalscope.api.benchmark import BenchmarkMeta, DefaultDataAdapter
from evalscope.api.dataset import Sample
from evalscope.api.evaluator import TaskState
from evalscope.api.registry import register_benchmark


@register_benchmark(
    BenchmarkMeta(
        name='my_qa',
        pretty_name='My QA',
        dataset_id='/path/to/my_qa_dataset',
        subset_list=['default'],
        metric_list=['exact_match'],
        eval_split='test',
        prompt_template='Answer the question.\n\nQuestion: {question}\nAnswer:',
    )
)
class MyQAAdapter(DefaultDataAdapter):

    def record_to_sample(self, record) -> Sample:
        return Sample(
            input=record['query'],
            target=record['answer'],
            metadata={'id': record.get('id', '')},
        )

    def extract_answer(self, prediction: str, task_state: TaskState) -> str:
        return prediction.strip()
```

如果模型经常输出解释，可以改成抽最后一行：

```python
def extract_answer(self, prediction: str, task_state: TaskState) -> str:
    lines = [line.strip() for line in prediction.splitlines() if line.strip()]
    return lines[-1] if lines else ''
```

### 10.3 需要自定义评分的任务

如果你的答案不是简单字符串比较，比如：

```text
模型输出 JSON，要求字段完全正确
模型输出代码，要求运行测试
模型输出多个标签，要求 F1
模型输出自由文本，要求 LLM judge
```

就重写 `match_score()`。

示例：检查 JSON 字段：

```python
import json
from evalscope.api.evaluator import TaskState
from evalscope.api.metric import Score


def match_score(
    self,
    original_prediction: str,
    filtered_prediction: str,
    reference: str,
    task_state: TaskState,
) -> Score:
    score = Score(
        extracted_prediction=filtered_prediction,
        prediction=original_prediction,
    )

    try:
        pred_obj = json.loads(filtered_prediction)
        ref_obj = json.loads(reference)
        passed = pred_obj == ref_obj
    except Exception as e:
        passed = False
        score.metadata['error'] = str(e)

    score.value = {'acc': passed}
    score.main_score_name = 'acc'
    return score
```

### 10.4 只想换 prompt，不想写新 benchmark

可以优先看 `dataset_args` 是否能覆盖 `prompt_template`、`subset_list`、`few_shot_num` 等 metadata。

Python 方式：

```python
from evalscope import run_task
from evalscope.config import TaskConfig

task_cfg = TaskConfig(
    model='qwen-onnx',
    eval_type='openai_api',
    api_url='http://127.0.0.1:8000/v1/chat/completions',
    api_key='EMPTY',
    datasets=['arc'],
    dataset_args={
        'arc': {
            'subset_list': ['ARC-Easy'],
            'prompt_template': 'Answer with only the letter.\n\n{question}\n\n{choices}',
        }
    },
    limit=5,
    work_dir='outputs_prompt_test',
)

run_task(task_cfg)
```

如果只是实验 prompt，尽量先用 `dataset_args`。如果数据格式、答案抽取、评分逻辑都变了，再新写 adapter。

## 11. 新增 evaluation 的决策树

```text
我要评测一个新数据集
  |
  +-- 是 A/B/C/D 选择题？
  |     |
  |     +-- 是 -> 继承 MultiChoiceAdapter
  |              通常只写 record_to_sample()
  |
  +-- 是普通文本答案？
  |     |
  |     +-- 是 -> 继承 DefaultDataAdapter
  |              写 record_to_sample()
  |              必要时写 extract_answer()
  |
  +-- 分数不是简单字符串比较？
  |     |
  |     +-- 是 -> 重写 match_score()
  |
  +-- 需要多个 subset？
  |     |
  |     +-- 在 BenchmarkMeta.subset_list 里列出来
  |
  +-- 需要 few-shot？
  |     |
  |     +-- 设置 train_split 和 few_shot_num
  |     +-- 必要时重写 sample_to_fewshot()
  |
  +-- 需要特殊聚合？
        |
        +-- 设置 aggregation 或重写 aggregate_scores()
```

## 12. 添加 evaluation 的实际步骤

### Step 1: 找一个最像的现有 adapter

选择题看：

```text
evalscope/benchmarks/arc/arc_adapter.py
evalscope/benchmarks/commonsense_qa/commonsense_qa_adapter.py
```

代码任务看：

```text
evalscope/benchmarks/humaneval/humaneval_adapter.py
```

普通问答/复杂任务可以全局搜：

```bash
rg "class .*Adapter\\(DefaultDataAdapter\\)" evalscope/evalscope/benchmarks
```

### Step 2: 新建 adapter 文件

比如：

```text
evalscope/evalscope/benchmarks/my_eval/my_eval_adapter.py
```

### Step 3: 写 `BenchmarkMeta`

你最少要决定：

```text
name
dataset_id
subset_list
metric_list
eval_split
prompt_template
```

### Step 4: 写 `record_to_sample()`

这是最关键的一步。

你要把原始数据统一成：

```text
input: 给模型看的题目主体
choices: 选择题选项，可选
target: 标准答案
metadata: 样本 id、类别、额外字段
```

### Step 5: 写答案抽取

如果模型输出格式不稳定，重写 `extract_answer()`。

### Step 6: 写评分

简单任务用 `metric_list`。复杂任务重写 `match_score()`。

### Step 7: 确保 benchmark 被 import

注册装饰器只有在 Python import 这个模块时才会执行。

如果你写了文件但 `--datasets my_eval` 提示 benchmark not found，通常是模块没有被 import。

排查：

```bash
rg "my_eval" evalscope/evalscope/benchmarks
rg "import.*adapter|benchmarks" evalscope/evalscope/benchmarks evalscope/evalscope -g '__init__.py'
```

### Step 8: 小样本跑通

先只跑 1-5 条：

```bash
evalscope eval \
  --model qwen-onnx \
  --api-url http://127.0.0.1:8000/v1/chat/completions \
  --api-key EMPTY \
  --eval-type openai_api \
  --datasets my_eval \
  --limit 5 \
  --work-dir outputs_my_eval_debug \
  --debug
```

### Step 9: 看三类文件

```text
logs/eval_log.log
predictions/<model>/<dataset>_<subset>.jsonl
reviews/<model>/<dataset>_<subset>.jsonl
```

先确认：

```text
prompt 是否正确？
模型输出是否合理？
extract_answer 抽出来的值是否正确？
reference 是否正确？
score 是否正确？
```

不要一开始就跑全量。新增 evaluation 最常见的 bug 都在 prompt、字段映射、答案抽取。

## 13. 常见问题

### 13.1 为什么 `evalscope eval --model .` 会找 safetensors？

因为没有指定 `--eval-type openai_api` 时，EvalScope 会把 `.` 当成本地 Hugging Face checkpoint。

你的 Olive 输出是 ONNX Runtime GenAI 包，不是 PyTorch safetensors checkpoint。所以要：

```bash
--eval-type openai_api
--api-url http://127.0.0.1:8000/v1/chat/completions
```

### 13.2 `subset_list` 是什么意思？

一个 benchmark 可以有多个子集。ARC 有：

```python
subset_list=['ARC-Easy', 'ARC-Challenge']
```

这代表一次 `--datasets arc` 会分别评测：

```text
arc / ARC-Easy
arc / ARC-Challenge
```

最后 report 里会有两个 subset 的分数。

如果你只想跑一个：

```python
dataset_args={
    'arc': {
        'subset_list': ['ARC-Easy']
    }
}
```

### 13.3 模型回答很多文字，EvalScope 怎么知道对错？

流程是：

```text
原始回答 -> filter_prediction() -> extract_answer() -> metric
```

选择题里 `MultiChoiceAdapter.extract_answer()` 会尽量从文本里找 A/B/C/D。

所以 prompt 通常会要求：

```text
The entire content of your response should be:
ANSWER: [LETTER]
```

这不是必须，但能大幅减少答案抽取失败。

### 13.4 Dashboard 没有东西显示

通常原因是路径填错或 reports 还没生成。

Dashboard 要填：

```text
outputs_onnx_api
```

不是：

```text
outputs_onnx_api/20260626_094333
```

并且目录里必须有：

```text
outputs_onnx_api/<timestamp>/reports/<model>/<dataset>.json
```

### 13.5 `predictions` 和 `reviews` 有什么区别？

```text
predictions:
  模型原始输出缓存。
  用来避免重复调用模型。

reviews:
  评分结果缓存。
  包含抽取后的答案、reference、score。
  用来避免重复评分。
```

如果你改了 `extract_answer()` 或 `match_score()`，旧 predictions 仍可复用，但 reviews 应该重新生成。

## 14. 你真正需要记住的设计

EvalScope 的核心设计是三层：

```text
Model layer:
  ModelAPI / OpenAICompatibleAPI / LazyModel
  只负责“给 messages，返回 completion”

Benchmark layer:
  DataAdapter / BenchmarkMeta / Sample
  负责“数据集怎么变成 prompt，输出怎么变成答案”

Evaluator layer:
  DefaultEvaluator / CacheManager / Report
  负责“并发跑、缓存、聚合、写报告”
```

新增 evaluation 主要改 Benchmark layer。

除非你要支持一种新的模型服务协议，否则不用动 Model layer。

除非你要改变整个运行编排方式，否则不用动 Evaluator layer。

对你当前 ONNX + OpenAI API 场景来说：

```text
ONNX wrapper 属于 Model layer 外部服务。
EvalScope 的 OpenAICompatibleAPI 只是 HTTP client。
你要评测新数据集，主要写 DataAdapter。
```

## 15. 推荐阅读顺序

按这个顺序读源码比较不乱：

1. `evalscope/benchmarks/arc/arc_adapter.py`
   先看一个完整 benchmark 最小实现。

2. `evalscope/api/benchmark/adapters/multi_choice_adapter.py`
   看选择题 prompt 和答案抽取。

3. `evalscope/api/benchmark/adapters/default_data_adapter.py`
   看通用数据加载、推理、评分 hooks。

4. `evalscope/evaluator/evaluator.py`
   看整体评测编排、缓存、并发、聚合。

5. `evalscope/run.py`
   看 CLI 配置如何进入 evaluator。

6. `evalscope/models/openai_compatible.py`
   看 API 模型怎么发请求。

7. `evalscope/api/registry.py`
   看 benchmark/model/metric/evaluator 怎么注册。

如果目标是“我自己加一个 evaluation”，前 3 个文件最重要。
