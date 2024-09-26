# AutoGen 教程笔记

## 是什么？

https://github.com/microsoft/autogen

AutoGen 是微软开源的**多 agent 对话框架**，可以方便的构建 LLM 工作流，AutoGen **代理是可定制**的, **可对话的**, 并且可以**无缝地与人类参与**。 它们可以**使用 LLM, 人类输入, 和工具(python code)的各种模式运行**。

## 解决什么问题？

1. 简化了复杂 LLM 工作流的编排
2. 实现 LLM 应用自动化
3. 增强了 LLM 推理
4. 它支持多样化的对话模式

![AutoGen](https://static.trumandu.top/yank-note-picgo-img-20240925133702.png)

## Getting Started

### 安装

可以直接安装 AutoGen `pip install pyautogen` 也可以安装 UI 开发工具 AutoGen Studio,这里为了更快熟悉 AutoGen，可以考虑使用 UI 开发工具

https://microsoft.github.io/autogen/docs/autogen-studio/getting-started

1.创建 Python 环境

```
conda create -n autogen python=3.12
conda activate autogen
```

2.安装依赖

```
pip install autogenstudio
```

这一步会自动安装依赖 pyautogen

3.启动 UI 页面

```
autogenstudio ui --port 8888
```

如果有报错，请安装`pip install flaml[automl]`

![Img](https://static.trumandu.top/yank-note-picgo-img-20240925134746.png)

### 入门 Demo

1.创建 model

在 build 模块选中 model 创建一个 ollam api model ,当然你也可以创建一个 openai model

![Img](https://static.trumandu.top/yank-note-picgo-img-20240925135142.png)

2.创建 user_proxy 与 default_assistant autogen
在 agent 模块下创建一个 UserProxy Agent ,可以不用添加 model,但要开启高级选项中的 Code Execution Config 为 local，也就是说允许 agent 在本地执行代码。

![Img](https://static.trumandu.top/yank-note-picgo-img-20240925135520.png)

再创建一个 default_assistant ，添加之前创建的 model,System Message 设置如下：

```
You are a helpful AI assistant.
Solve tasks using your coding and language skills.
In the following cases, suggest python code (in a python coding block) or shell script (in a sh coding block) for the user to execute.
    1. When you need to collect info, use the code to output the info you need, for example, browse or search the web, download/read a file, print the content of a webpage or a file, get the current date/time, check the operating system. After sufficient info is printed and the task is ready to be solved based on your language skill, you can solve the task by yourself.
    2. When you need to perform some task with code, use the code to perform the task and output the result. Finish the task smartly.
Solve the task step by step if you need to. If a plan is not provided, explain your plan first. Be clear which step uses code, and which step uses your language skill.
When using code, you must indicate the script type in the code block. The user cannot provide any other feedback or perform any other action beyond executing the code you suggest. The user can't modify your code. So do not suggest incomplete code which requires users to modify. Don't use a code block if it's not intended to be executed by the user.
If you want the user to save the code in a file before executing it, put # filename: <filename> inside the code block as the first line. Don't include multiple code blocks in one response. Do not ask users to copy and paste the result. Instead, use 'print' function for the output when relevant. Check the execution result returned by the user.
If the result indicates there is an error, fix the error and output the code again. Suggest the full code instead of partial code or code changes. If the error can't be fixed or if the task is not solved even after the code is executed successfully, analyze the problem, revisit your assumption, collect additional info you need, and think of a different approach to try.
When you find an answer, verify the answer carefully. Include verifiable evidence in your response if possible.
Reply "TERMINATE" in the end when everything is done.

```

![Img](https://static.trumandu.top/yank-note-picgo-img-20240925135839.png)

3.创建 Default Workflow
![Img](https://static.trumandu.top/yank-note-picgo-img-20240925135923.png)

4.运行
![Img](https://static.trumandu.top/yank-note-picgo-img-20240925134746.png)

该 demo 对话流程如下：
![Img](https://static.trumandu.top/yank-note-picgo-img-20240925140103.png)

## 主要组件与功能

### 对话终止

1. initiate_chat 参数 max_turns=2，只回答两次
2. 代理触发的终止 max_consecutive_auto_reply 和 is_termination_msg

### 对话模式

1. Two-agent chat: 一对一对话
2. Sequential chat：按序列对话
3. Group Chat：群聊，从中按模式选择其中一个对话

### 人工输入模式

1. NEVER
2. TERMINATE
3. ALWAYS

### 代码执行

代码支持本地执行和 docker 执行。

在对话中使用代码执行

```
import tempfile

from autogen import ConversableAgent
from autogen.coding import LocalCommandLineCodeExecutor

# Create a temporary directory to store the code files.
temp_dir = tempfile.TemporaryDirectory()

# Create a local command line code executor.
executor = LocalCommandLineCodeExecutor(
    timeout=10,  # Timeout for each code execution in seconds.
    work_dir=temp_dir.name,  # Use the temporary directory to store the code files.
)
code_executor_agent = ConversableAgent(
    "code_executor_agent",
    llm_config=False,  # Turn off LLM for this agent.
    code_execution_config={"executor": executor},  # Use the local command line code executor.
    human_input_mode="NEVER",
)
code_writer_system_message = """You are a helpful AI assistant.
Solve tasks using your coding and language skills.
In the following cases, suggest python code (in a python coding block) or shell script (in a sh coding block) for the user to execute.
1. When you need to collect info, use the code to output the info you need, for example, browse or search the web, download/read a file, print the content of a webpage or a file, get the current date/time, check the operating system. After sufficient info is printed and the task is ready to be solved based on your language skill, you can solve the task by yourself.
2. When you need to perform some task with code, use the code to perform the task and output the result. Finish the task smartly.
Solve the task step by step if you need to. If a plan is not provided, explain your plan first. Be clear which step uses code, and which step uses your language skill.
When using code, you must indicate the script type in the code block. The user cannot provide any other feedback or perform any other action beyond executing the code you suggest. The user can't modify your code. So do not suggest incomplete code which requires users to modify. Don't use a code block if it's not intended to be executed by the user.
If you want the user to save the code in a file before executing it, put # filename: <filename> inside the code block as the first line. Don't include multiple code blocks in one response. Do not ask users to copy and paste the result. Instead, use 'print' function for the output when relevant. Check the execution result returned by the user.
If the result indicates there is an error, fix the error and output the code again. Suggest the full code instead of partial code or code changes. If the error can't be fixed or if the task is not solved even after the code is executed successfully, analyze the problem, revisit your assumption, collect additional info you need, and think of a different approach to try.
When you find an answer, verify the answer carefully. Include verifiable evidence in your response if possible.
Reply 'TERMINATE' in the end when everything is done.
"""

code_writer_agent = ConversableAgent(
    "code_writer_agent",
    system_message=code_writer_system_message,
    llm_config={"config_list": [{"model": "gpt-4", "api_key": os.environ["OPENAI_API_KEY"]}]},
    code_execution_config=False,  # Turn off code execution for this agent.
)
chat_result = code_executor_agent.initiate_chat(
    code_writer_agent,
    message="Write Python code to calculate the 14th Fibonacci number.",
)
```

### 工具使用

在 AutoGen Studio 中叫 Skills，在 AutoGen 中叫工具，两者都是 python 函数。也就是说在 AutoGen 可以使用自定义函数解决一些任务。

工具必须至少向**两个代理注册**，才能在对话中发挥作用。通过 register_for_llm 使用工具签名注册的代理可以调用该工具;通过 register_for_execution 向工具的 function 对象注册的代理可以执行工具的函数。

```
# Register the tool signature with the assistant agent.
assistant.register_for_llm(name="calculator", description="A simple calculator")(calculator)

# Register the tool function with the user proxy agent.
user_proxy.register_for_execution(name="calculator")(calculator)
```

或者

```
from autogen import register_function

# Register the calculator function to the two agents.
register_function(
    calculator,
    caller=assistant,  # The assistant agent can suggest calls to the calculator.
    executor=user_proxy,  # The user proxy agent can execute the calculator calls.
    name="calculator",  # By default, the function name is used as the tool name.
    description="A simple calculator",  # A description of the tool.
)
```

## 参考

1. [AutoGen](https://microsoft.github.io/autogen/docs/Getting-Started)
