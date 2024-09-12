# AI 学习

## 待学列表

1. LLM 微调
2. RAG
3. Agent
4. LangChain 使用

大模型优化顺序：零样本(Zero-shot)、少样本(Few-shot)推理 > LLM 微调

prompt工程中增加一些示例就是典型的few-shot learning.

模型训练分为：Pre-training,Post-training

## OpenAI fine-tuning

## 开源LLM 微调

LLM 微调是一个**有监督学习过程**，主要使用标注数据集来更新 LLM 的权重，并使模型提高其特定任务的能力。

Post-training 包含: Supervised Fine-Tuning(sft),RLHF,DPO,KTO

-   **Supervised Fine-Tuning(sft)** (监督微调)是一种在预训练模型上使用小规模有标签数据集进行训练的方法。 相比于预训练一个全新的模型，对已有的预训练模型进行监督微调是更快速更节省成本的途径。
-   **RLHF** (Reinforcement Learning from Human Feedback) 以通过人类反馈来进一步微调模型，使得模型能够更好更安全地遵循用户指令。分为两个：Reward model 和 PPO（Proximal Policy Optimization 近端策略优化）
-   DPO 基于人类偏好训练
-   KTO (Kahneman-Taversky Optimization)

常见微调框架：llama-factory、deepspeed、unsloth

-   [Lora 微调](https://zhuanlan.zhihu.com/p/650197598)
-   [QLoRA]()
-   [P-Tuning]()
-   [LLaMA Factory](https://github.com/hiyouga/LLaMA-Factory)
-   [unsloth](https://github.com/unslothai/unsloth)


## RAG

## Agent

## 参考

1. [LLaMA-Factory](https://llamafactory.readthedocs.io/zh-cn/latest/getting_started/installation.html)
2. [Fine-tuning - OpenAI API](https://openai.xiniushu.com/docs/guides/fine-tuning)
