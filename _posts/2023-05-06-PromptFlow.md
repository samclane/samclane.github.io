---
layout: post
comments: true
published: true
title: PromptFlow - FlowCharts for Chaining LLMs
description: Chain together multiple LLM calls to supercharge GPT
---

![PromptFlow](..\images\promptflow\heroscreenshot.png)

# PromptFlow

LLM applications are currently a rats-nest of LLM calls, with no way to visualize the flow of the application. Additionally, people are using GPT to things like make HTTP calls and interact with databases, all of which add even more complexity to the application. [PromptFlow](http://www.promptflow.org) is a tool to help visualize the flow of your LLM application, and to help you chain together multiple LLM calls in a more user-friendly way.


## How PromptFlow Works

PromptFlow is built on a visual flowchart editor, making it simple to create nodes and connections between them. Each node can represent a prompt, a Python function, or an LLM. Connections between nodes define conditional logic, allowing you to craft the flow of your program seamlessly.

When you execute your flowchart, PromptFlow runs each node according to the order defined by the connections, passing data between nodes as required. If a node returns a value, that value is automatically passed to the next connected node in the flow.

## Initial Setup

Getting started with PromptFlow is easy. First, ensure you have Python 3.9 or higher installed. Then, install the required dependencies with the following command:

```bash
python -m pip install -r requirements.txt
```

## Launching PromptFlow

To launch PromptFlow, simply run the following command:

```bash
python run.py
```

If you encounter any issues, double-check that your PYTHONPATH is set correctly:

```bash
export PYTHONPATH=$PYTHONPATH:.
```

## Usage

To use an LLM, we'll introduce 3 nodes- the [`LLM`](LLM), [`Prompt`](Prompt), and [`History`](History) Nodes. Let's make a chat with a caveman. First, build the following flowchart:

![image](..\images\promptflow\caveman1.png)

Note the cycle at the end of the chart. This will allow us to carry on our conversation with the caveman.

Next, we need to give our AI a prompt to act as a caveman. Double click on the lower [`Prompt`](Prompt) label on the Prompt node to open the prompt editor. Fill out the `Label` and [`Prompt`](Prompt) as follows:

![image](..\images\promptflow\caveman2.png)

Then, hit `File -> Save` or `Ctrl+S` to save the prompt. You should see the prompt appear in the flowchart:

![image](..\images\promptflow\caveman3.png)

Now press `Run`, or `F5` to run the flowchart. Let's ask the caveman who George Washington is. You should see the output of each node in the console on the right:

```text
Init: 

[System: Done]
Start: 
Prompt: You are a caveman, answer each question as such. Ooga booga.

History: You are a caveman, answer each question as such. Ooga booga.

Input: who was george washington?
History: who was george washington?
LLM: Me not know who George Washington is. Me caveman, not know much about outside world.
Input: None

[System: Done]
```

That's good, as a caveman probably wouldn't know who George Washington is. Let's ask him about rocks:

```text
[System: Already initialized]
Start: 
Prompt: You are a caveman, answer each question as such. Ooga booga.

History: You are a caveman, answer each question as such. Ooga booga.

Input: what's your favorite kind of rock?
History: what's your favorite kind of rock?
LLM: Me like shiny rock. Shiny rock pretty.
Input: None

[System: Done]
```

Note the system doesn't initialize again, as it's already been initialized.

## Documentation

For a comprehensive guide to using PromptFlow, check out our official documentation at [promptflow.org](promptflow.org).

## Contributing to PromptFlow

We welcome and encourage contributions to PromptFlow! You can contribute by building a node or helping us improve the existing ones. If you come across any bugs or issues, please don't hesitate to create an issue, open a pull request, or let us know in our [Discord](https://discord.com/invite/5MmV3FNCtN) server.
