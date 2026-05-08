---
name: chinese-llm-research
description: 调研中国大模型（DeepSeek、智谱GLM、通义Qwen等）的基准数据、模型信息、技术规格的标准方法论
category: research
---

# 中国大模型调研方法论

## 何时使用
当需要对比、查找中国大模型（DeepSeek、智谱GLM、通义Qwen、月之暗面Kimi等）的基准数据、模型信息、技术规格时使用。

## 关键坑点

### 1. 模型名称可能不准确
- **DeepSeek V4** 不存在，最新是 **DeepSeek V3**（2024年12月发布）
- **GLM 5.1** 实际产品名为 **GLM-5**，API平台下拉选项中有"GLM-5.1"
- 调研前先在官网确认实际模型名称

### 2. 搜索工具对中国厂商模型几乎无效
- `search_files` 在 `hongkong.chat` 等来源上命中率极低
- 必须改用 `browser_navigate` 直接访问官方页面

### 3. GLM系列Benchmark数据极难获取
- 智谱官方几乎不公布具体数字，只声称"达到SOTA"
- GLM-5 的 SWE-bench、HumanEval 具体分数**从未在任何官方文档中公开**
- 开源版本 GLM-4-32B-0414（THUDM/zai-org GitHub）有部分数据
- 对比时要明确标注"声称SOTA，无公开数字"

### 4. DeepSeek数据相对透明
- 官方GitHub README有完整benchmark表格
- Raw地址：`https://raw.githubusercontent.com/deepseek-ai/DeepSeek-V3/main/README.md`
- 可以通过 `browser_console` 在页面提取表格数据

## 推荐调研路径

1. **DeepSeek** → GitHub README（完整数据）+ 官网API文档
2. **智谱GLM** → GitHub (THUDM/zai-org) + bigmodel.cn + zhipuai.cn
3. **通义Qwen** → GitHub (QwenLM) + dashscope
4. **月之暗面Kimi** → 官网，通常不发布技术报告

## 验证清单
- [ ] 确认模型实际名称（避免用V4查V3）
- [ ] 所有数字必须来自官方一手来源（非营销页面的文字声称）
- [ ] 区分"开源版本"和"API专有版本"的数据差异
- [ ] GLM系列要特别注明"无公开具体数字"
