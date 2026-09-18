+++
date = '2026-09-18T22:20:00+08:00'
draft = false
title = 'LLM、Agent二三事'
+++

> AI预警！昇腾预警！胡说八道警告！

## 前言

近来折腾了两件事。

一件是在一台昇腾 910B4 服务器上（单机双卡）把本地大模型跑起来，另一件是拿它去搭一个 agent。两件事都不算特别成功，也不算失败，就是过程挺有意思，记一记。

## 一、先把模型跑起来

### 硬件与环境

手上的资源不多，就一台服务器，插了两张 910B4，每张 32G，加起来 64G 的显存。听起来挺充裕，实际上要开长上下文、还要留 KV cache 的空间，就得精打细算了。

- 部署形态：单服务器、双卡
- 卡：昇腾 910B4 32G × 2
- 推理框架：[vllm-ascend](https://github.com/vllm-project/vllm-ascend)（镜像 `quay.io/ascend/vllm-ascend:v0.23.0`）
- 模型：Qwen3.8-27B-w8a8
- 部署方式：docker compose，容器直接用 host 网络

### 部署配置

直接上 compose 文件，这份是我现在正在跑的：

```yaml
services:
  vllm-ascend:
    image: quay.io/ascend/vllm-ascend:v0.23.0
    container_name: vllm-ascend-qwen3.8
    restart: unless-stopped

    # host 网络模式（HCCL 多卡通信必需）
    network_mode: host

    # 共享内存 1GB（文档推荐值）
    shm_size: 1g

    # NPU 设备映射
    devices:
      - /dev/davinci2
      - /dev/davinci3
      - /dev/davinci_manager
      - /dev/devmm_svm
      - /dev/hisi_hdc

    volumes:
      - /usr/local/dcmi:/usr/local/dcmi:ro
      - /usr/local/Ascend/driver/tools/hccn_tool:/usr/local/Ascend/driver/tools/hccn_tool:ro
      - /usr/local/bin/npu-smi:/usr/local/bin/npu-smi:ro
      - /usr/local/Ascend/driver/lib64/:/usr/local/Ascend/driver/lib64/:ro
      - /usr/local/Ascend/driver/version.info:/usr/local/Ascend/driver/version.info:ro
      - /etc/ascend_install.info:/etc/ascend_install.info:ro
      - /root/.cache:/root/.cache
      - ./Qwen3.8-27B-w8a8:/root/Qwen3.8-27B-w8a8

    environment:
      - VLLM_USE_MODELSCOPE=False
      - PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
      - HCCL_BUFFSIZE=1024
      - TASK_QUEUE_ENABLE=1
      - OMP_PROC_BIND=false
      - OMP_NUM_THREADS=4
      - VLLM_API_KEY=xxxx
    # 启动命令（基于文档 Qwen3.5-27B-w8a8 部署脚本）
    command: >
      vllm serve /root/Qwen3.8-27B-w8a8
      --host 0.0.0.0
      --port 8000
      --data-parallel-size 1
      --tensor-parallel-size 2
      --quantization ascend
      --served-model-name qwen3.8
      --block-size 128
      --max-num-seqs 16
      --max-model-len 131072
      --max-num-batched-tokens 16384
      --trust-remote-code
      --enable-prefix-caching
      --gpu-memory-utilization 0.90
      --speculative_config '{"method": "mtp", "num_speculative_tokens": 3, "enforce_eager": true}'
      --compilation-config '{"cudagraph_mode":"FULL_DECODE_ONLY"}'
      --additional-config '{"enable_cpu_binding":true}'
      --reasoning-parser qwen3
      --enable-auto-tool-choice
      --tool-call-parser qwen3_coder

    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 360s
```

启动后就是个标准的 OpenAI 兼容接口，`http://<host>:8000/v1`，`/health` 探活，跟平时用 vllm 没什么两样。

### 几个关键参数

这份配置是在官方部署脚本的基础上做过精简的，剩下的都是必须项，挑几个关键的说说：

- `--tensor-parallel-size 2` 和 `--quantization ascend`：双卡必须 TP=2——27B 的权重即使 w8a8 量化过，单卡 32G 也塞不下权重加 KV cache；量化走昇腾官方这条路最省心。
- `--max-model-len 131072`：128K 上下文。真要顶满，KV cache 会吃掉不少显存，得看实际负载。
- `--max-num-seqs 16` 和 `--max-num-batched-tokens 16384`：并发压得比较保守，主要是给长上下文让路，稳定比吞吐重要。
- `--enable-prefix-caching`：多轮对话里前缀重复率高，开了之后体感明显。
- `--speculative_config` 里的 MTP：投机解码，`num_speculative_tokens: 3`，算是白捡的速度，代价是显存多占一点。
- `--reasoning-parser qwen3` 和 `--tool-call-parser qwen3_coder`：一个把思考过程拆成单独字段，一个用来接 agent 的工具调用。

## 二、选型：从 qwen3.8-27b 出发，绕一圈又回来

### 起点：一开始就定了它

选型其实没怎么纠结。一开始就定了 qwen3.8-27b——在单机双卡这个硬件配置里，据闻它是性能最好的那个。

- dense 架构，27B 的参数量，w8a8 量化后显存占用正好落在"两张 32G 卡能舒服放下"的区间里；
- 能力上比小模型明显强一截，日常写代码、读文档、梳理逻辑基本够用；
- 128K 的上下文能开起来，长文档、大文件不至于切得七零八落；
- 支持 MTP 投机解码，速度这个短板也补上了；
- 官方对 qwen3 系列的 reasoning parser 和 tool call parser 支持都跟得上，省心。

27B 这个尺寸，在这个档位的硬件上，算是那个「甜点位」：不用纠结，直接用。

### 翻车：输出大 JSON 时，它不吐字了

用着用着，就撞上了一个很尴尬的问题。

有一个场景，需要模型输出一段比较大的 JSON 参数。结果就是：**vllm 那边很久都没有吐出数据，最后请求超时失败。**

不是报错，不是崩溃，就是安静。像极了开会时被点名的人。

问了 AI，说是 vllm 在输出 tool call 参数的时候，会攒着完整的参数再一次性吐出来，所以大 JSON 这种场景就翻车了——参数没攒齐之前，前面什么都不吐，客户端的超时先到，请求就废了。这个说法不一定对，但现象是稳定的、可复现的。

于是就有了换个思路的念头。

### 双小模型：一个卡一个，各管一摊

我当时的想法是，既然大模型在"长时间不出字"这件事上翻车，那不如拆开——两张卡分别跑一个小模型，一个负责一摊事，单个请求的输出短一点、响应快一点，就不容易超时了。

方案是两个单卡小模型：

1. qwen3.5-9b，做需要世界知识的分析。
2. minicpm5-2b，快快地干活。

单卡跑得动，部署起来也轻松，理论上确实解决了"又慢又久"的问题。

但很快就发现另一个更朴素的事实：**小模型还是智商不足啊。**

- 通用问答还凑合；
- 需要知识时，就感觉没那么智能；
- 尤其 minicpm5-2b 写查询 SQL，经常写错——字段名对不上、条件写反、语法缺东西，你纠正它一次，它下次还能错得不一样。

省下的那点响应时间，全在"反复纠正、反复重试"里还回去了。而且重试本身也要时间，最后的总耗时并不比大模型好，体验还更差。

### 回到 27b，把思考降下来

兜兜转转，还是回到了 qwen3.8-27b。

这次换了个思路：既然问题出在"想太久、说得太慢"，那就不让它想那么久。**把思考等级降到 low。**

这个改动带来的变化挺明显：

- 输出明显干脆了，不再动辄一整屏的思考过程；
- 整体响应快了一截；
- 代价是复杂推理上的深度会有点损失，但对这个场景来说，够用就行。

但要说清楚：**降思考只是提速，真正把那个大 JSON 的问题解决掉，靠的还是工程化。**

思路是优化 agent 的流程——不再让它最后吐一个大 JSON，而是把前面的每一块工作先缓存起来，最后不用数据本身，只用各块的 id 来组装。具体的做法放在后面 agent 那一节说。

绕了一圈，结论是：**模型大小不是可以随便换的旋钮，把它换小，省下的是算力，付出的是正确率。**

## 三、Agent：重点不在模型，在工具

模型跑起来之后，下一步自然就是拿它搭 agent。

### 为什么是 eino

框架选了 [eino](https://github.com/cloudwego/eino)，云原生那边开源的 Go LLM 应用框架。

理由很直接：我们后端就是 Go 的，用它不用再跨语言折腾，模型、工具、编排都在一套代码里。

### session、turn、step

用之前先补了点概念课，简单说说我的理解：

- **session**：一次会话的生命周期。里面有历史消息、有状态，跨多个回合持续存在。
- **turn**：一轮交互，从用户输入开始，到最终给出回答结束。
- **step**：turn 内部的一次原子动作。一次模型调用是一个 step，一次工具调用也是一个 step。

一个 turn 往往由多个 step 串起来：模型思考 → 决定调用工具 → 工具返回 → 模型再看结果 → 决定下一步。理清这层结构之后，最大的好处是**排障有着落了**：效果不好时，能分清到底是模型判断错了、工具给的信息不对、还是编排的流程本身有问题，而不是笼统地说"这 agent 不行"。

### 心得：用工具去制约模型，而不是指望模型

折腾下来，我最大的一点体会是：

**至少在小的 LLM 上，靠"把工具设计好"来制约、或者说提效 agent，比指望模型变聪明见效快得多。**

模型的能力是固定的，你改不动。但工具是你定义的，你可以让它"难以被用错"。

### 具体做法：别让 LLM 写 SQL，让它填 AST

最有代表性的是前面那个写 SQL 的场景。

一开始的做法很直觉：让 LLM 自主探测表结构，自己写 SQL 语句。结果就是小模型各种翻车，大模型也不是永远稳。

后来的做法是：**不要直接让 LLM 写 SQL 了，而是提供一个类似 AST 构建器的东西。**

思路是把"写 SQL"这件事拆开：

- 代码这边定义好一套结构化的参数：查哪张表、选哪些字段、什么条件、什么操作符、怎么聚合、怎么排序、分页多少；
- 让 AI 只做"填空"，输出这些结构化参数；
- 工具内部再把这份结构转成 AST，最后生成 SQL。

这么一改，效果立刻不一样：

- **格式管控在 tool 里**。模型不需要记 SQL 方言的细节，也不需要拼字符串，出错的地方少了；
- **错误可以被结构化地拒绝**。字段名不存在、操作符不合法，tool 直接拦下来，返回明确的错误信息，模型照着改就行，而不是给它一句语法错误的 SQL 让它自己猜；
- **能力边界清楚了**。哪些能做、哪些不能做，是由工具定义的，不由模型的发挥决定。

说白了，这就是把复杂性从模型搬到了代码里。写 SQL 本来就是个"填空 + 拼装"的问题，那就让模型只负责填空这一步它擅长的部分。

### 回到那个大 JSON：拆开、缓存、只传 id

前面说的大 JSON 超时，最后不是靠降思考强度解决的，而是靠改 agent 的流程。

原来的流程是：让模型一路做下来，最后一次性吐出所有参数。问题就出在这个"最后一次性"上——前面的工作成果全压在一个超大 JSON 里，输出又长又慢，还容易翻车。

改法是把流程拆开：

- 让前面的每一块工作先做、先落缓存，各产出各自的结果，不需要攒在一起；
- 最后组装时，不要再把一大坨数据塞回模型，而是只传各块的 id，让工具按 id 去取缓存里的内容做拼装。

这样一来，模型的输出从"一个大 JSON"变成了"若干个小的、单纯的调用"，每次要说的东西都很少，超时的风险自然就没了。而且中间结果有缓存，也能复用、能回看，出错了能定位到是哪一块。

说到底还是那句话：**把复杂性从模型搬到代码里。**

### 一点延伸

写到这里忽然想到：把模型围起来、给它定好工具和流程的这一整套，好像就是大家说的 harness 吧？难道这就是 harness 么——不太确定，先这么叫。

## 四、效果

没做过压测，也没什么正经的 benchmark。目前基本没有并发，就是我自己在对话和 agent 里用。

就这个体感来说，输出还是不错的：

- 日常问答、读代码找问题，基本不用怎么催，能给出可用的答案；
- 长文档喂进去，上下文不会明显掉链子；
- 开了前缀缓存加投机解码之后，多轮对话的等待感明显下降。

至于并发能力，等后续多用起来再说吧，现在说都是空话。

## 胡思乱想

这两件事折腾下来，有几个感受。

**昇腾的生态也不错。** vllm-ascend 的镜像和文档都做得不错，基本看官网的文档抄一份能跑的配置，再慢慢改。我这份 compose，基本也是这么来的。

**小模型加好工具，有时候比大模型加裸提示更靠谱。** 至少在我这个场景里，"降低模型难度"比"提升模型能力"要现实得多。

**AI 时代程序员的活，好像越来越像"设计接口"了。** 模型负责填，我们负责定义填什么、怎么校验、错了怎么办。这个活看起来简单，其实挺讲究。

## 后话

这次的部署，坦白说，绝大部分是照着文档和社区脚本抄+改出来的，我自己干的活主要是「敢于尝试」和「耐心等待编译」。

下一步打算把 agent 那块再打磨打磨，把工具设计得更"笨"一点，看看小场景下能不能真的跑顺。到时候再写一篇。

---

PS. 我说的「输出还是不错的」，是自用标准，不是生产标准。真要上生产，还是得压测、得验证，别拿这篇当验收依据。

PS. 让模型填空，比让它自由创作，靠谱多了。这句话放在人身上好像也成立。
