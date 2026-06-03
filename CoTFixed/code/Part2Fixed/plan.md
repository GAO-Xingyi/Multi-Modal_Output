# Part2Fixed 工作流方案

## 0. 前置背景

- **Part1（已完成）**：把原始 CoT 中所有"用字符拼凑画出的结构图"区域识别出来，按出现顺序以 `<smiles_use>id</smiles_use>` 占位符标注。产出形如 [result.txt](../Part1Located/exp/DBLocated/prompt/result.txt) 中"标记以后的 CoT"。
- **Part2（本计划）**：把每个 `<smiles_use>id</smiles_use>` 真正翻译为可机读的 SMILES 并回填，得到清洗后的慢思考语料。

输入资产：
- `marked_cot`：Part1 的输出，含占位符的整段 CoT。
- 可选 `original_cot`：Part1 的输入。Part1 替换掉的 ASCII 拼图本身**通常无法直接还原成可靠的分子**（拼得太残、缺信息、还伴随作者反复"哦不对"自我推翻），所以 Part2 的主信号**不来自原始 ASCII**，而来自占位符附近作者用自然语言写下的物质名 / 类别描述。

---

## 1. 用户原始计划（澄清后）

> 占位符附近能提取到的上下文不多，但作者通常已经在文本里写出了"是什么物质"，例如：
>
> > `…结构简式就是饱和六元碳环：<smiles_use>5</smiles_use>（六元环，环己烷的结构）`
>
> 这里要为 id=5 提取的"对象"是 **"饱和六元碳环 / 六元环 / 环己烷"** 这些自然语言关键词。把这些关键词 + 短上下文拼进 prompt 给功能模型，让模型据此输出对应的 SMILES，再回填回去。

整体方向是对的，但和我之前理解的"对象=原始 ASCII 片段"完全不同。重新组织如下。

---

## 2. 点评

### 2.1 优点
- **以自然语言关键词为锚点**而非以残破的 ASCII 拼图为锚点，是更符合数据现实的选择。原始拼图常残缺到无法独立判读，但作者几乎总会在前后文用一句中文/化学式补丁说明"这里其实是 XX"。这句话才是高信噪比信号。
- 翻译目标聚焦在 SMILES 单一形态，简单清晰，不再纠结多种输出。
- 占位符回填仍是确定性字符串替换，风险低。

### 2.2 需要明确 / 容易踩坑的点

1. **"对象提取"的范围与抽取方式要定死**
   样本里同一个 id 的物质名可能出现在前面（"结构就是饱和六元环…"）也可能出现在后面（"…（六元环，环己烷的结构）"），还可能两边都有但说法不同（甚至前后矛盾，因为作者经常自我推翻）。建议：
   - 取占位符 **前 N₁ 字、后 N₂ 字** 作为粗上下文（N₁≈200, N₂≈150 起步）。
   - 由功能模型负责从这段上下文里挑关键词并最终给出 SMILES，**不要**再单独搞一个抽取模型，多一道反而引入误差累积。换句话说"对象提取"这一步并入"翻译"一步做，prompt 里显式要求模型先列出它识别到的物质关键词、再给 SMILES，便于人审。

2. **窗口里的其他占位符不展开**
   上下文窗口可能跨到相邻 id（result.txt 里 id=4/5/6 就紧挨着）。规则：窗口里碰到 `<smiles_use>k</smiles_use>` 保持原样不展开，让模型知道"那是另一处占位符，不归你处理"。

3. **作者反复自我推翻怎么办**
   CoT 风格特征：作者会反复"不对，应该是…"。同一个 id 在窗口里可能先说"苯环"后说"环己烷"。规则：**取离占位符最近的、未被推翻的描述**。让 prompt 显式说明这一点，并要求模型把"采用了哪一句作为依据"写进输出，便于排查。

4. **置信度与失败兜底**
   - 当窗口里完全没有可识别的物质名（极端情况），模型返回 `confidence=low`，`smiles=null`。
   - 回填策略：高/中置信 → 写入 SMILES；低置信 → 保留占位符不替换并落入 `failures.json` 等人工处理。**绝不让模型瞎编 SMILES**。

5. **输出格式建议**
   推荐回填为带属性的标签，下游既能渲染也能保留 id：
   ```
   <smiles_use id="5" smiles="C1CCCCC1" name="环己烷"/>
   ```
   而不是裸 SMILES。这样将来想改渲染策略只改下游解析器，不必重跑模型。

6. **可重入与缓存**
   每个 id 的翻译结果单独落盘（`translations/{cot_id}/{smiles_id}.json`），断点可续。

7. **评测**
   抽 20~50 个 id 做人工标注 SMILES，统计：
   - SMILES 正确（同分异构判同）的比例；
   - 模型 fallback (low) 的比例；
   - 模型自信但错（hallucinate）的比例 ← **最关键的指标**，必须接近 0。

---

## 3. 完善后的方案（推荐版）

### 阶段 A：上下文窗口构造（确定性，非 LLM）
对每个占位符 `<smiles_use>{id}</smiles_use>`：
- 在 `marked_cot` 中定位；
- 截取前 N₁=200、后 N₂=150 字符；
- 不展开窗口里其它 `<smiles_use>k</smiles_use>`；
- 落盘 `windows/{id}.json`，字段：`{id, window, anchor_offset_in_window}`。

> 这一步是纯字符串切片，不要让模型做。

### 阶段 B：翻译（LLM，单 id 一次调用）
prompt 模板要点：
```
SYSTEM:
你是有机化学专家。下面给你一段 CoT 上下文窗口，里面有一个占位符 <smiles_use>{id}</smiles_use>，
作者本想在那里画出一个有机分子结构，但因为是文本输出无法画图。请根据窗口里
作者用自然语言或化学式提到的物质身份信息，推断该处对应的分子，并以严格 JSON 输出。

规则：
- 仅判定 id={id} 这一个占位符。窗口里出现的其它 <smiles_use>k</smiles_use> 与你无关，按字面忽略。
- 作者的 CoT 常自我推翻；以**离占位符最近且未被作者明确否定**的描述为准。
- 不确定就给 confidence=low、smiles=null，禁止编造。

USER:
[窗口]
{window}

输出 JSON：
{
  "id": {id},
  "evidence": "你采用的依据原句（直接引用窗口中的文字）",
  "name_zh": "物质中文名或类别（如 环己烷 / 饱和六元环）",
  "smiles": "C1CCCCC1 或 null",
  "confidence": "high" | "med" | "low"
}
```
建议带 3~5 个 few-shot：苯、环己烷、对苯二酚、丙烯酸甲酯链状双键、苯甲酸。

### 阶段 C：回填（确定性，非 LLM）
- `confidence ∈ {high, med}` → 把 `<smiles_use>{id}</smiles_use>` 替换为
  `<smiles_use id="{id}" smiles="{smiles}" name="{name_zh}"/>`。
- `confidence == low` → 不替换，把 id 加入 `failures.json`。
- 替换前断言占位符在 `marked_cot` 中**恰好出现一次**。

### 阶段 D：自检
- 把 `final_cot` 中所有 `<smiles_use ... />` 标签反向替换为 `<smiles_use>{id}</smiles_use>` 后，应与 `marked_cot` 完全 byte-equal。
- 每条 CoT 输出统计：处理 id 数 / high / med / low 占比。

### 阶段 E：抽样人评
- 随机抽 20~50 个 id，人工写参考 SMILES；
- 计算正确率、low 比例、hallucinate（high/med 但错）比例；
- hallucinate 比例不达标 → 调强模型或扩窗口。

---

## 4. 与原计划差异速记

| 维度 | 你原计划的理解（可能） | 修正后 |
|---|---|---|
| 对象提取的来源 | 不明确，但听起来像把原 ASCII 拼图也送进去 | 主信号是窗口内**自然语言里的物质名**；ASCII 拼图通常没用甚至有害，可以不送 |
| 上下文提取与对象提取 | 两步分开 | 合成一步交给同一个 LLM，避免误差累积；但要求模型先输出 evidence/name 再给 SMILES |
| 输出形态 | "翻译" | 严格 JSON，含 evidence + smiles + confidence |
| 不识别时 | 未定义 | 显式 fallback：保留占位符 + failures.json |
| 回填 | 模型或脚本 | 必须脚本确定性替换 + 唯一性校验 |

---

## 5. 目录结构建议
```
Part2Fixed/
├── plan.md                       # 本文件
├── prompt/
│   └── translateV1.txt           # 阶段 B 的 prompt 模板
├── src/
│   ├── build_window.py           # 阶段 A
│   ├── translate.py              # 阶段 B
│   ├── backfill.py               # 阶段 C
│   └── verify.py                 # 阶段 D
├── data/
│   ├── windows/{cot_id}.json
│   ├── translations/{cot_id}.json
│   ├── final/{cot_id}.txt
│   └── failures.json
└── eval/
    ├── samples.jsonl
    └── report.md
```

## 6. 推荐的下一步
1. 用 [result.txt](../Part1Located/exp/DBLocated/prompt/result.txt) 里那 6 个 id 手工跑一遍阶段 A，确认窗口大小够用、没切断关键句。
2. 起草 `translateV1.txt`，先单条跑 id=5（"环己烷"是简单 case，便于验证 happy path），再跑 id=1/2/3（同一物质多次出现，验证模型是否稳定一致）。
3. 跑完 6 个 id，人工核验，再决定窗口参数与是否需要更强模型。
4. **不要**急着把 ASCII 原文也塞 prompt——先验证仅靠自然语言关键词能覆盖多少比例，再决定是否需要把 ASCII 作为补充信号。
