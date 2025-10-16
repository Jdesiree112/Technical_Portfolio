# Prompting
[Prompt engineering](https://www.ibm.com/think/topics/prompt-engineering) is the craft of tuning a model response through natural language, directly telling and showing the model how to perform a given task. When engineering a prompt, the goal is to strike a balance between complexity and completeness, avoiding excessive instruction while remaining thorough. 

The difficulty with this is the fact that AI models initially perform better as instruction increases, followed by a point of diminishing returns, then degraded responses.[1](https://arxiv.org/html/2402.14848v1) This instance is not particularly well understood, but it is evident when testing prompts to be true. Techniques such as CoT have been effective in mitigating degradation, but not preventing it. With this, the goal becomes striking a balance, optimizing prompts to minimize token input while providing adequate structure. 

This goal can be met through the utilization of known and newly discovered prompting patterns and strategies, which is what I will discuss below.

## Content
This document covers prompting techniques, patterns, and strategies as well as some use cases. Examples covered:

**Example-Based Learning:**
- Few-Shot

**Reasoning & Thought Process Frameworks:**
- Chain-of-Thought (CoT)
- Tree-of-Thought (ToT)
- Thread-of-Thought (ThoT)
- ReAct (Reason and Act)

**Self-Analysis & Verification Techniques:**
- Self-Ask
- Self-Consistency
- Step-Back
- Reflection
- Meta

**Context & Identity Framing:**
- Role-playing
- Persona
- Domain Priming
- Contextual Priming

### Example-Based Learning
- **[Few-Shot Prompting](https://www.promptingguide.ai/techniques/fewshot)**<br>
Few-Shot Prompting is a basic form of prompt engineering where ideal response examples are supplied as context. This serves many roles in a prompt's ability to refine responses, as the examples will instruct the model on personality, formatting, content structuring, and inclusion/exclusion parameters.
  - This method is effective in many applications, including but not limited to: classification, summarization, and   reformatting/structuring of data.
  - While effective in some cases, this method is not ideal when dealing with reasoning, creative, or nuanced tasks.

### Reasoning & Thought Process Frameworks
- **[Chain-of-Thought (CoT)](https://www.promptingguide.ai/techniques/cot)**
CoT is an industry-recognized strategy for handling complex reasoning tasks by guiding the model to take intermediate reasoning steps. This type of prompting has been shown to reduce length-induced failure and generally increase the accuracy of responses. Creating a CoT prompt involves identifying atomic steps in a process, similar to know a human may plan to approach a complex task, defining what is to be done in each step. Steps are ordered logically so that each benefits those that come after it. Often, few-shot prompting is integrated into CoT prompts to refine the processing of individual steps or the output in general.
  - This method is ideal for complex, multi-step reasoning or multi-task prompts.
  - Using CoT for simple, single-step tasks is not ideal as it is not likely to yield significant improvements. Few-shot examples or general instructions are better suited for simple tasks. Another notable issue is that CoT is subject to fact hallucination.
 
- **[Chain-of-Dictionary (CoD)](https://learnprompting.org/docs/advanced/few_shot/chain-of-dictionary)**


- **[Cue-CoT](https://learnprompting.org/docs/advanced/few_shot/cue-based-chain-of-thought)**


- **[Chain of Knowledge (CoK)](https://learnprompting.org/docs/advanced/few_shot/chain-of-knowledge)


- **[Tree-of-Thought (ToT)](https://www.promptingguide.ai/techniques/tot)**
ToT is a strategy that mimics the architecture of a decision tree, building on the concept of CoT. ToT was developed for tasks that require exploration or strategic lookahead. The idea is that the model engages in problem-solving through tree search via a multi-round conversation, or may be used in an agentic agent workflow where a panel of specialized agents discusses a topic, then evaluates the full conversation among them to derive an answer.
  - This strategy is best used in cases where strategic decision-making involves assessing multiple possibilities creatively. Unlike CoT, where a single possibility is approached, ToT allows for more creativity.
  - When real-time, step-by-step action and immediate self-correction based on immediate results are the primary need (in these cases, methods like ReAct may be more suitable).
  
- **[Thread-of-Thought (ThoT)](https://learnprompting.org/docs/advanced/thought_generation/thread_of_thought?srsltid=AfmBOooTUTfjMTfhx9e1pvHO2GF3-19yBgQq8pOe-8NeiaabzIisEn0Q)**
ThoT offers an alternative to previously existing methods such as **Retrieval-Augmentation with Long Context Extensions** and **Prompt Streamlining**, mitigating the need for fine-tuning. Similar to other reasoning-focused prompts, this strategy implements guided step-by-step processing but with a focus on analysing and summarizing parts of chaotic contexts. The model is instructed through inclusion and exclusion parameters to filter irrelevant information out of long context entries. The overall prompt structure is designed in two passes: first analysis and second summarization with derived conclusions.
  - This strategy is most notably praised for effectiveness with RAG outputs, which may be chaotic and have varying relevance. Ideally, especially in RAG implementation, this strategy would be paired with intention analysis.
  - ThoT is not designed for simple tasks or refined input tasks.
  
- **[ReAct (Reason and Act)](https://www.promptingguide.ai/techniques/react)
ReAct is a well-known prompting framework where the model is instructed to generate both reasoning traces and task-specific actions in an interleaved manner. This approach draws on the concept of self-reflection, where the model iteratively assesses the task, then reiterates with the help of its previous reasoning, and few-shot prompting, where it learns from examples. When creating a ReAct Prompt, you issue the model the task/question and a series of thoughts followed by actions, closing with a thought. The model then produces the action for the last thought.
  - ReAct is particularly useful in multi-step reasoning and planning, forming a ReAct prompt with instructions for the model to replicate the process given in context until it reaches a conclusion. This may also be beneficial in reducing face hallucination or processing multi-step processes, such as in building agents or chatbots that can handle complex customer inquiries or technical issues by reasoning through the problem, utilizing relevant tools (e.g., knowledge bases, diagnostic tools), and providing step-by-step solutions.
  - While powerful, ReAct prompting has flaws. Notably, it struggles when encountering non-informative results from RAG, resulting in it becoming off track in reasoning. It struggles to recover, if at all. Additionally, the structural constraints reduce its flexibility in formulating reasoning steps.
 
### Self-Analysis & Verification Techniques
- **[Self-Ask](https://learnprompting.org/docs/advanced/few_shot/self_ask?srsltid=AfmBOoqWZcIZV8xVIb4VgaK3xyL1jIg2I2xvGj_hBv-70sYULpxCHHiO)**
Self-ask prompting involves instructing the model to break complex questions into small sub-questions and answering them step-by-step. The concept behind the development of those strategies was derived from the idea of an LLM asking itself follow-up questions, in effect, building off of its own thinking.
  - This strategy is generally employed in cases such as customer support, legal analysis, research, and creative writing, where it may be integrated with tools such as web search.
  - Self-ask is limited in its effectiveness, where the benefit of using it hinges on the model's ability to create effective  sub-questions, and it struggles to manage abstract queries. Each application of this strategy can be accomplished by other means, such a ToT to drive creative results through a reasoning process.

- **[Self-Consistency](https://www.promptingguide.ai/techniques/consistency)**
Self-consistency prompting was developed for the purpose of replacing the naive greedy decoding used in chain-of-thought prompting, sampling multiple reasoning paths through few-shot CoT. The process uses generations to select the most consistent answer among many. The theory is that out of many answer's the most common would be correct, similar to how you may assume the most common answer in a poll is correct.
  - This strategy is used mainly for reasoning tasks and improving accuracy, with some cases being for fact-checking.
  - Self-consistency is ineffective for certain types of free-form generation and requires a high computational cost versus single generations.
    
- **[Reflection](https://www.linkedin.com/pulse/day-16-reflection-prompting-teaching-ai-self-evaluate-gupta-uvyye/)**
Reflection processing is when you ask a model to reflect on its own response, analyzing it before finalizing. This is a strategy I often include at the end of complex reasoning prompts as part of my closing instructions. It is known for having a self-learning effect, with the model revising its output. An effective reflection prompt can be as simple as "review your work for any missing points." It is best paired with complex prompts where accuracy is critical.
  - Reflection is useful in combination with another strategy, acting as a follow-up rather than a standalone prompt. It may be sent alone as a follow-up turn to correct errors in the previous generation.
  - Using reflection can, at times, cause hallucinations. This is because many models are prone to treating user input as fact, ignoring that what it said was correct and assuming SOMETHING is wrong. This can be mitigated by adding conditionality to the reflection statement, such as "Correct X if it doesn't align with Y. If X aligns with Y, then make no changes." This prompts the model to check, but does not require action.

- **[Meta](https://www.promptingguide.ai/techniques/meta-prompting)**
Meta-prompting is a prompt engineering technique where a prompt is used to generate, refine, or interpret another prompt. This can be thought of as prompting to generate a prompt, focusing on structure and form rather than content. This is often used to refine prompts for a specific task or domain.
  - This strategy is useful for workflow optimization or reordering content in a prompt you wrote. For example, let's say you have a base prompt for integration into a customer service agent, and you want the content in the prompt to be structured so that dependencies feed into one another in a more logical way. Maybe you should have an analysis of the user query before you begin the process of generating a response. You would be able to include the prompt along with structuring guidance. This can also be used to optimize user prompts in a workflow, preprocessing the prompt so that it is structured effectively before being passed to the model or a RAG pipeline.
  - This is not necessary for simple tasks.

**In Progress**
**Context & Identity Framing:**
- Role-playing
- Persona
- Domain Priming
- Contextual Priming






