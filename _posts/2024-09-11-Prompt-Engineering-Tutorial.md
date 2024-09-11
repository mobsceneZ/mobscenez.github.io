---
layout: post
title:  "提示词工程的常用技巧"
tags: [Prompt Engineering]
---

虽然之前一直揶揄许多工作实质上是在做提示词工程(**prompt engineering**)，但是当自己的工作也被要求和大语言模型结合的时候人就老实了。这篇博客根据**Anthropic**的[Prompt Engineering Interactive Tutorial](https://github.com/anthropics/courses)总结当下最流行的提示词技巧。

### 1. 清晰明了地描述指令

请将大模型视作新接手工作的人，除你告诉它该做的事外它没有任何关于此事的上下文信息。因此，你将你所想达成的目标描述得越清晰直观，大模型的回复就越准确。

如果你不确定你的提示词是否准确，可以向你的同事或朋友展示它并确认他们能否根据指令得到预期的结果。如果他们感到困惑，那么大模型也如此。

### 2. 分配角色

有时候提示大模型充当特定角色是很有帮助的，这被称作角色提示(**role prompting**)。角色提示类似告知人们**像...一样去思考**并且能改变大模型回复的风格、说辞和方式。

此外，可向模型提供有关目标受众的上下文，例如“你是一只猫”和“你是一只跟滑板爱好者沟通的猫”对大模型的影响是不同的。

### 3. 分离数据与指令

很多时候，我们并不想书写完整的提示词，而是期望一个能反复修改以提供额外输入数据的提示词模板(**prompt template**)。为此，我们可以将提示词中的固定骨架从可变用户输入中分离，并按需替换用户输入的部分。

例如，如果要求大模型输出不同动物的声音，可以将提示词模板设置为：
```python
# Variable content
ANIMAL = "Cow"
# Prompt template with a placeholder for the variable content
PROMPT = f"I will tell you the name of an animal. Please respond with the noise that animal makes. {ANIMAL}"
```

在引入占位符变量时，请明确标出变量开始和结束的位置。为此，可将输入用`XML`标签包装，形如`<tag>content</tag>`。诸如`Claude`的大模型被特意训练来识别提示词组织结构中的`XML`标签。

### 4. 格式化输出

你可以要求大模型按照你期望的格式输出内容。此外，`Claude`等大模型也支持回复补全功能，即预先设置部分模型回答并让模型在此回答基础上输出剩余内容：
```python
# Claude Messages API
message = client.messages.create(
    model = "claude-3-haiku-20240307",
    max_tokens = 2000,
    messages = [
        {"role": "user", "content": "Please write a haiku about Cat. Use JSON format with the keys as \"first_line\", \"second_line\", and \"third_line\"."},
        {"role": "assistant", "content": "{"}
    ]
)
```

### 5. 一步步思考

如果有人直接摇醒你让你马上回答若干复杂问题，你的脑子肯定是嗡嗡的。我们需要时间来思考，大模型也同理。允许大模型一步步思考问题有时能让其更准确(但这些思考必须显式地表达出来)，例如：
```python
# Prompt
PROMPT = "Name a famous movie starring an actor who was born in the year 1956. First brainstorm about some actors and their birth years in <brainstorm> tags, then give your answer."
```

### 6. 少样本提示词

少样本提示词(**few-shot prompting**)是指给予大模型部分示例来告知其应该怎么做和不应该怎么做，这往往非常实用，例如：
```python
PROMPT = """Please complete the conversation by writing the next line, speaking as "A".
Q: Is the tooth fairy real?
A: Of course, sweetie. Wrap up your tooth and put it under your pillow tonight. There might be something waiting for you in the morning.
Q: Will Santa bring me presents on Christmas?"""
```
你当然可以花时间来思考要求大模型以怎样的语气或格式完成任务，但为大模型提供若干现成、格式良好的例子或许更加简单。

### 7. 避免幻觉

模型幻觉是指其可能会声明不存在或不正确的事物，为缓解这一问题，可以采取如下方法：
- 允许模型不知道关于某一问题的答案。
- 要求模型在回答前给出确切证据。
- 适当调低`temperature`值以生成更确定的回答。

### 8. 从头构建复杂提示词

这里展示`Anthropic`为复杂提示词精心设计的结构，其包含许多提示词工程元素。一般来说，最好先使用尽可能多的提示词元素来确保你的提示词正确工作，再精心雕琢和削减你的提示词。此外，提示词元素的先后顺序也会对回答质量产生影响，下面展示的顺序是不错的选择。
```python
######################################## INPUT VARIABLES ########################################

# First input variable - the conversation history (this can also be added as preceding `user` and `assistant` messages in the API call)
HISTORY = ""

# Second input variable - the user's question
QUESTION = ""



######################################## PROMPT ELEMENTS ########################################

##### Prompt element 1: `user` role
# Make sure that your Messages API call always starts with a `user` role in the messages array.
# The get_completion() function as defined above will automatically do this for you.

##### Prompt element 2: Task context
# Give Claude context about the role it should take on or what goals and overarching tasks you want it to undertake with the prompt.
# It's best to put context early in the body of the prompt.
TASK_CONTEXT = ""

##### Prompt element 3: Tone context
# If important to the interaction, tell Claude what tone it should use.
# This element may not be necessary depending on the task.
TONE_CONTEXT = ""

##### Prompt element 4: Detailed task description and rules
# Expand on the specific tasks you want Claude to do, as well as any rules that Claude might have to follow.
# This is also where you can give Claude an "out" if it doesn't have an answer or doesn't know.
# It's ideal to show this description and rules to a friend to make sure it is laid out logically and that any ambiguous words are clearly defined.
TASK_DESCRIPTION = ""

##### Prompt element 5: Examples
# Provide Claude with at least one example of an ideal response that it can emulate. Encase this in <example></example> XML tags. Feel free to provide multiple examples.
# If you do provide multiple examples, give Claude context about what it is an example of, and enclose each example in its own set of XML tags.
# Examples are probably the single most effective tool in knowledge work for getting Claude to behave as desired.
# Make sure to give Claude examples of common edge cases. If your prompt uses a scratchpad, it's effective to give examples of how the scratchpad should look.
# Generally more examples = better.
EXAMPLES = ""

##### Prompt element 6: Input data to process
# If there is data that Claude needs to process within the prompt, include it here within relevant XML tags.
# Feel free to include multiple pieces of data, but be sure to enclose each in its own set of XML tags.
# This element may not be necessary depending on task. Ordering is also flexible.
INPUT_DATA = f""

##### Prompt element 7: Immediate task description or request #####
# "Remind" Claude or tell Claude exactly what it's expected to immediately do to fulfill the prompt's task.
# This is also where you would put in additional variables like the user's question.
# It generally doesn't hurt to reiterate to Claude its immediate task. It's best to do this toward the end of a long prompt.
# This will yield better results than putting this at the beginning.
# It is also generally good practice to put the user's query close to the bottom of the prompt.
IMMEDIATE_TASK = ""

##### Prompt element 8: Precognition (thinking step by step)
# For tasks with multiple steps, it's good to tell Claude to think step by step before giving an answer
# Sometimes, you might have to even say "Before you give your answer..." just to make sure Claude does this first.
# Not necessary with all prompts, though if included, it's best to do this toward the end of a long prompt and right after the final immediate task request or description.
PRECOGNITION = ""

##### Prompt element 9: Output formatting
# If there is a specific way you want Claude's response formatted, clearly tell Claude what that format is.
# This element may not be necessary depending on the task.
# If you include it, putting it toward the end of the prompt is better than at the beginning.
OUTPUT_FORMATTING = ""

##### Prompt element 10: Prefilling Claude's response (if any)
# A space to start off Claude's answer with some prefilled words to steer Claude's behavior or response.
# If you want to prefill Claude's response, you must put this in the `assistant` role in the API call.
# This element may not be necessary depending on the task.
PREFILL = ""



######################################## COMBINE ELEMENTS ########################################

PROMPT = ""

if TASK_CONTEXT:
    PROMPT += f"""{TASK_CONTEXT}"""

if TONE_CONTEXT:
    PROMPT += f"""\n\n{TONE_CONTEXT}"""

if TASK_DESCRIPTION:
    PROMPT += f"""\n\n{TASK_DESCRIPTION}"""

if EXAMPLES:
    PROMPT += f"""\n\n{EXAMPLES}"""

if INPUT_DATA:
    PROMPT += f"""\n\n{INPUT_DATA}"""

if IMMEDIATE_TASK:
    PROMPT += f"""\n\n{IMMEDIATE_TASK}"""

if PRECOGNITION:
    PROMPT += f"""\n\n{PRECOGNITION}"""

if OUTPUT_FORMATTING:
    PROMPT += f"""\n\n{OUTPUT_FORMATTING}"""
```