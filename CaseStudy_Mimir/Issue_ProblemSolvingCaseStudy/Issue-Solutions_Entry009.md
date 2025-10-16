# Prompt Library Architecture Redesign

## Overview

This application has evolved beyond its original narrow scope into a comprehensive educational AI system. The original implementation utilized a monolithic ML-driven prompt classification system embedded within a broad prompt library, creating architectural complexity that impedes maintainability and scalability.

The existing ML classification approach, while initially effective, has accumulated significant technical debt and faces scalability limitations in a single-developer environment. The original monolithic prompt library embedded within `app.py` has become unwieldy for iterative development and prompt refinement.

To address these architectural constraints, the system is undergoing a fundamental redesign that replaces the ML classification tree with a sophisticated agentic workflow. This new architecture delegates prompt selection and task management to specialized lightweight agents, each handling specific aspects of the educational interaction process. Concurrently, the prompt library has been extracted into a dedicated `prompt_library.py` module with granular, purpose-specific prompt segments.

The agentic workflow approach provides deterministic, auditable decision-making while eliminating the maintenance overhead, retraining cycles, and technical debt associated with the previous ML classification system. This refactoring significantly improves code organization, facilitates rapid prompt iteration, and enables scalable system expansion without the complexity of managing ML model lifecycles.

The new architecture combines lightweight decision agents with a sophisticated thinking agent utilizing a custom-trained Phi3 model for complex reasoning preprocessing, creating a more maintainable, transparent, and capable educational AI system.

### Practice Question System Enhancement
The practice question framework underwent significant restructuring:

**`STRUCTURE_PRACTICE_QUESTIONS`** was redesigned to explicitly define formatting rules for three distinct components of practice question construction:
1. Question formulation and context
2. Data reference integration (tables, charts, tool outputs)
3. Answer format specification (single choice, multiple selection, written response)

Additionally, conditional rules were implemented for data interpretation-based questions that require visual or tabular analysis.

**`PRACTICE_QUESTION_FOLLOWUP`** was introduced to systematically address three possible user response scenarios:
- Correctly answered questions (congratulations and progression options)
- Incorrectly answered questions (constructive feedback and explanation)
- Unanswered questions (encouragement and assistance offers)

### Formatting System Modularization
Formatting rules were extracted from individual prompts and recomposed into two specialized modules:
- **`GENERAL_FORMATTING`**: Universal formatting guidelines for content organization and structure
- **`LATEX_FORMATTING`**: Mathematical notation and LaTeX-specific rendering instructions

This separation was implemented because LaTeX formatting is only required for mathematical instruction contexts. In non-mathematical scenarios, LaTeX instructions constitute unnecessary token overhead for the response generation system.

### Specialized Cognitive Prompts for Thinking Agent
A sophisticated thinking agent system has been introduced, utilizing a custom-trained Phi3 model to handle high-complexity reasoning tasks. Three specialized cognitive prompts have been designed for the thinking agent:

- **`MATH_THINKING`**: Implements Tree-of-Thought processes for mathematical problem-solving and quantitative reasoning guidance
- **`QUESTION_ANSWER_DESIGN`**: Provides structured approaches to question formulation and response design using Chain-of-Thought methodologies
- **`REASONING_THINKING`**: Delivers logical reasoning and critical thinking instruction frameworks through Chain-of-Thought processes

These prompts are designed to work independently or in combination, removing cognitive burden from the response model by preprocessing complex reasoning requirements through a dedicated thinking agent.

### Core Educational Prompts
Additional specialized prompts support the core educational functionality:
- **`GUIDING_TEACHING`**: Core pedagogical guidance for educational interactions

### Tool Decision Engine Prompt Centralization
The prompt previously embedded within the `TinyLlama/TinyLlama-1.1B-Chat-v1.0` tool decision engine has been centralized within the prompt library. This prompt, designated as **`TOOL_DECISION`**, was extracted from the main `app.py` and integrated into `prompt_library.py` within its own labeled section. The variable reference now replaces the previously hardcoded prompt text, ensuring consistent prompt management across all system components.

This centralization provides:
- Unified prompt version control and management
- Consistent editing and iteration processes across all system prompts
- Improved maintainability and reduced code duplication
- Clear separation between business logic and prompt content

### Conversation History Integration
All context-dependent and scenario-based prompts were enhanced with explicit instructions for conversation history utilization, ensuring consistent consideration of prior interactions in response generation.

## Problem Analysis: Classification System Misalignment

The current ML classifier was trained on the original prompt taxonomy, which included segments that have since been deprecated or restructured during optimization. This creates a fundamental misalignment between the trained model's output space and the current prompt architecture.

The core architectural question emerges: Should the system adapt the classification model to accommodate the new prompt segmentation, or should alternative routing mechanisms be implemented?

## Solution Analysis

This segment discusses options that were considered, weighing benefits and drawbacks for each. 

### ML Classifier Retraining

Retraining the existing classifier to support the refined prompt taxonomy would maintain the current ML-driven approach while adapting to architectural changes.

#### Benefits
- **Preserves nuanced intent interpretation**: ML classifiers excel at handling ambiguous user inputs and contextual subtleties that rule-based systems struggle with
- **Maintains existing architecture**: Minimal changes to the current LangGraph workflow and routing logic
- **Proven effectiveness**: The current classifier has demonstrated reliable performance with the previous prompt library
- **Handles edge cases**: ML models can interpolate between training examples to handle novel input patterns

#### Drawbacks
- **Training data dependency**: Requires comprehensive dataset generation, labeling, and validation for each prompt modification
- **Deployment complexity**: Model updates necessitate retraining pipelines, model versioning, and A/B testing infrastructure
- **Technical debt accumulation**: Each prompt library modification creates a retraining obligation, coupling content changes to ML infrastructure
- **Scalability limitations**: Adding new prompt segments requires exponential growth in training data to cover interaction combinations
- **Maintenance overhead**: Ongoing model performance monitoring, drift detection, and retraining scheduling

### Pure Logical Rules

Implementing deterministic routing logic would replace ML classification with explicit conditional statements based on observable input characteristics.

#### Benefits
- **Deterministic behavior**: Predictable routing decisions that can be traced and debugged systematically
- **Rapid iteration**: Prompt additions require only logical rule modifications, eliminating retraining cycles
- **Transparent decision making**: Rule logic is explicitly defined and auditable, facilitating troubleshooting
- **Resource efficiency**: Eliminates ML inference overhead and model storage requirements
- **Version control simplicity**: Rule changes are directly visible in code diffs and can be reviewed like standard code
- **Granular segment control**: Well-suited for the new highly specific prompt segments that map to observable patterns

#### Drawbacks
- **Limited contextual interpretation**: Rule-based systems struggle with ambiguous inputs that require nuanced understanding
- **Rule complexity explosion**: Comprehensive coverage may require extensive conditional logic that becomes difficult to maintain
- **Brittleness to edge cases**: Hard-coded rules may fail gracefully when encountering unexpected input patterns
- **Loss of learning capability**: Cannot adapt to new patterns without explicit programming
- **Maintenance burden shift**: Complexity moves from ML pipeline maintenance to rule logic maintenance

### Hybrid Architecture

A hybrid approach would preserve ML classification for high-level intent recognition while using logical rules for deterministic segment selection.

#### Benefits
- **Optimal specialization**: Leverages ML for ambiguous intent interpretation and logic for deterministic decisions
- **Preserved investment**: Maintains the current working classifier without requiring immediate retraining
- **Incremental scalability**: New prompt segments can be added through logical rules without ML pipeline modifications
- **Fault tolerance**: ML failures can fall back to logical rules, and vice versa
- **Performance optimization**: Critical path decisions use appropriate mechanisms (fast logic vs. nuanced ML)
- **Segment-appropriate routing**: ML handles discovery vs. teaching mode decisions while logic manages formatting, follow-ups, and conditional segments

#### Drawbacks
- **Architectural complexity**: Maintains two separate decision-making systems with potential interaction issues
- **Testing complexity**: Requires comprehensive integration testing for ML-logic interactions
- **Debugging challenges**: Failures may stem from either system, complicating troubleshooting
- **Performance overhead**: Both ML inference and rule evaluation introduce latency
- **Maintenance burden**: Requires expertise in both ML operations and rule-based system maintenance

### LangChain Template Integration

Implementing LangChain prompt templates would provide structured composition mechanisms for prompt segment assembly.

#### Benefits
- **Ecosystem integration**: Leverages existing LangChain infrastructure already present in the orchestration layer
- **Template reusability**: Prompt segments can be composed into different configurations for various scenarios
- **Conditional logic support**: Native template conditionals enable sophisticated prompt assembly patterns
- **Version control clarity**: Template modifications are explicitly tracked and reviewable
- **Testing granularity**: Individual template components can be unit tested independently
- **Structured composition**: Formal template syntax prevents ad-hoc string concatenation errors
- **Modular segment management**: Well-suited for managing the new granular prompt segments with clear dependency relationships
- **Cross-system consistency**: Templates can manage both response generation and tool decision prompts uniformly

#### Drawbacks
- **Abstraction overhead**: Additional layer between routing logic and final prompt generation
- **Learning curve**: Team must understand LangChain templating syntax and conventions
- **Performance impact**: Template parsing and variable substitution introduce computational overhead
- **Debugging complexity**: Template conditional logic can be more difficult to trace than explicit Python code
- **Dependency risk**: Increased coupling to LangChain ecosystem evolution and breaking changes
- **Template debugging**: Conditional template errors may be less transparent than string manipulation failures

## Implementation Plan

Based on the architectural analysis and considering the constraints of a single-developer project, the following implementation approach has been determined:

### Decision Rationale

The ML classifier, while effective in its original scope, will be removed due to mounting technical debt relative to benefits, maintenance overhead, response testing latency, and scalability limitations. The restructured prompt library's granular nature is better suited to targeted agent-based decisions combined with logical routing.

### Thinking Agent Architecture

A sophisticated preprocessing system utilizing a custom-trained Phi3 model will handle high-complexity reasoning tasks. This thinking agent represents a significant memory allocation commitment but is expected to dramatically improve accuracy, creativity, and application range. The custom model will be trained on a specific split:
- 20% `MATH_THINKING` applications
- 30% `REASONING_THINKING` applications  
- 20% individual instruction applications
- 30% combined instruction applications

Data will be curated where the full prompt as the model will see it is the first column, and the golden output will be in the second column. Prior to training the model, templates will be defined to ensure the training data reflects the model's expected input.

The model incorporates Tree-of-Thought processes for mathematical reasoning, Chain-of-Thought processes for general reasoning, enhanced domain knowledge in high school and undergraduate mathematics, and refined pedagogical processes.

### Thinking Agent Prompt Combinations

The thinking agent operates with specific combination rules:

1. **`MATH_THINKING`**: Triggered by regex logical process detecting mathematical terms in user input, applied alongside `LATEX_FORMATTING` in respective model processes

2. **`MATH_THINKING` + `QUESTION_ANSWER_DESIGN`**: Applied when regex expression returns true AND Agent 1 returns true for practice question necessity

3. **`REASONING_THINKING`**: Activated by two triggers - tool decision engine output equals true OR Agent 3 returns true for `PRACTICE_QUESTION_FOLLOWUP`

4. **`REASONING_THINKING` + `QUESTION_ANSWER_DESIGN`**: Allocated based on logical and agentic workflow process outputs using conditional Boolean value checking

### Prompt State Management System

A Python dictionary will manage the state of prompt segments for use by agents and routing logic. `CORE_IDENTITY` and `TOOL_DECISION` are excluded from state tracking as they are universally applied and would always evaluate to true.

**Default State Configuration:**
```python
library_state = {
    "MATH_THINKING": False,
    "QUESTION_ANSWER_DESIGN": False,
    "REASONING_THINKING": False,
    "VAUGE_INPUT": False,
    "USER_UNDERSTANDING": False,
    "GENERAL_FORMATTING": False,
    "LATEX_FORMATTING": False,
    "GUIDING_TEACHING": False,
    "STRUCTURE_PRACTICE_QUESTIONS": False,
    "PRACTICE_QUESTION_FOLLOWUP": False,
    "TOOL_USE_ENHANCEMENT": False,
}
```

All prompts begin each turn with false values, being updated by their respective logical or agentic processes. Upon response completion, values reset to default state for the next interaction cycle.

### Prompt Template Architecture

**Core Structure:**
- `CORE_IDENTITY` will be universally applied to all responses
- Formatting decisions will use logic-based determination:
  - `GENERAL_FORMATTING` alone for non-mathematical topics
  - `GENERAL_FORMATTING` + `LATEX_FORMATTING` when mathematical content is required
- `TOOL_USE_ENHANCEMENT` continues to be based on the existing tool decision engine output (YES = True, NO = False)

### Agentic Workflow Implementation

The system will employ specialized micro-agents performing focused tasks in conjunction, utilizing the same model as the tool decision engine (`TinyLlama/TinyLlama-1.1B-Chat-v1.0`) to avoid additional memory allocation overhead.

**Agent Specifications:**
- **Agent 1**: Determines practice question necessity based on current prompt and short-term conversation history (2 turns maximum)
- **Agent 2**: Evaluates whether `VAUGE_INPUT` or `USER_UNDERSTANDING` prompts are needed, analyzing current turn only (may output neither)
- **Agent 3**: Identifies follow-up scenarios based on current turn and 2-turn chat history, returning boolean value for `PRACTICE_QUESTION_FOLLOWUP`
- **Agent 4**: Assesses the need for `STRUCTURE_PRACTICE_QUESTIONS` and `GUIDING_TEACHING` prompts using current and recent conversation turns with a dedicated agent system prompt, utilizing the same lightweight model architecture

### Routing Logic for Core Educational Prompts

**`STRUCTURE_PRACTICE_QUESTIONS` Routing:**
Activated when Agent 4 determines that the conversation context and user input indicate readiness for structured practice question delivery, based on recent educational progression and topic comprehension indicators.

**`GUIDING_TEACHING` Routing:**
Triggered when Agent 4 identifies that the user requires direct pedagogical guidance rather than discovery-based learning, determined through analysis of current queries and conversation patterns indicating specific knowledge gaps or learning objectives.

### Response Integration Architecture

A logical function will process agent outputs and update the `library_state` dictionary, then construct the final system instruction through deterministic combination rules based on the activated prompt segments. The thinking agent will preprocess complex reasoning requirements and provide contextual support to the response generation model.

**Revised Prompt Template:**
```python
complete_prompt = f"""
{CORE_IDENTITY}
{prompt_segments}
If tools were used, the context and output will be here. Ignore if empty:
Image output: {tool_img_output}
Image context to consider: {tool_context}
Conversation history, if available:        
{recent_history}
Consider any context available to you:
{thinking_context}
Here is the user's current query:
{user_query}
        
"""
```

This template incorporates thinking context from the preprocessing agent, providing enhanced reasoning support while maintaining essential contextual elements and proper spacing for optimal model performance.

### Technical Benefits

This approach combines agentic tactics for nuanced decision-making with reliable coded logic for result integration, enhanced by sophisticated preprocessing through the thinking agent. The elimination of the ML classifier addresses the primary scalability and maintenance concerns while preserving the system's ability to make context-aware routing decisions through specialized, lightweight agents. The state management system provides clear tracking and debugging capabilities while ensuring consistent prompt activation across the entire system. The thinking agent significantly enhances the system's reasoning capabilities, accuracy, and creative range, justifying the increased memory allocation through substantially improved user experience.

## **Order of Operations**
This section details parts of the new system design, including the connections between agents and logical operations. 

### **1. System Initialization**
```python
# Initialize global state management
global_state_manager = GlobalStateManager()

# Define library_state dictionary (reset per turn)
library_state = {
    "MATH_THINKING": False,
    "QUESTION_ANSWER_DESIGN": False, 
    "REASONING_THINKING": False,
    "VAUGE_INPUT": False,
    "USER_UNDERSTANDING": False,
    "GENERAL_FORMATTING": False,
    "LATEX_FORMATTING": False,
    "GUIDING_TEACHING": False,
    "STRUCTURE_PRACTICE_QUESTIONS": False,
    "PRACTICE_QUESTION_FOLLOWUP": False,
    "TOOL_USE_ENHANCEMENT": False,
}
```

### **2. User Input Processing**
```python
# User submits prompt
user_input = get_user_input()

# Retrieve conversation history from global state
conversation_state = global_state_manager.get_conversation_state()
recent_history = format_recent_history(conversation_state['conversation_state'])
```

### **3. Tool Decision Engine**
```python
# Run tool decision engine (independent, uses user prompt only)
tool_decision_result = tool_decision_engine.should_use_visualization(user_input)

if tool_decision_result:
    tool_img_output, tool_context = generate_graph_tool(user_input)
    library_state["TOOL_USE_ENHANCEMENT"] = True
else:
    tool_img_output, tool_context = "", ""
```

### **4. Regex Logical Expressions**
```python
# General formatting is always applied
library_state["GENERAL_FORMATTING"] = True

# Mathematical keyword detection for LaTeX formatting
math_regex = r'\b(math|calculus|algebra|geometry|equation|formula|solve|calculate)\b'
if re.search(math_regex, user_input, re.IGNORECASE):
    library_state["LATEX_FORMATTING"] = True

# Additional regex checks update library_state...
```

### **5. Sequential Agent Execution**
```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.messages import SystemMessage, HumanMessage

# Initialize TinyLlama model for agents (already downloaded on CPU)
tinyllama_model = AutoModelForCausalLM.from_pretrained(
    "TinyLlama/TinyLlama-1.1B-Chat-v1.0",
    torch_dtype=torch.float16,
    device_map="cpu",
    trust_remote_code=True
)
tinyllama_tokenizer = AutoTokenizer.from_pretrained(
    "TinyLlama/TinyLlama-1.1B-Chat-v1.0",
    trust_remote_code=True
)

def run_agent_with_langchain(system_prompt, user_message, model, tokenizer):
    """Run agent using LangChain with TinyLlama"""
    messages = [
        SystemMessage(content=system_prompt),
        HumanMessage(content=user_message)
    ]
    
    # Format for TinyLlama
    formatted_prompt = ""
    for msg in messages:
        if isinstance(msg, SystemMessage):
            formatted_prompt += f"<|system|>\n{msg.content}<|end|>\n"
        elif isinstance(msg, HumanMessage):
            formatted_prompt += f"<|user|>\n{msg.content}<|end|>\n"
    formatted_prompt += "<|assistant|>\n"
    
    # Generate response
    inputs = tokenizer(formatted_prompt, return_tensors="pt")
    outputs = model.generate(
        **inputs,
        max_new_tokens=50,
        temperature=0.1,
        do_sample=True,
        pad_token_id=tokenizer.eos_token_id
    )
    
    response = tokenizer.decode(outputs[0], skip_special_tokens=True)
    return response.split("<|assistant|>")[-1].strip()

# Agent 1: Practice question necessity
agent_1_input = f"User Input: {user_input}\nRecent History: {recent_history[-4:]}"
agent_1_result = run_agent_with_langchain(
    system_prompt=agent_1_system,
    user_message=agent_1_input,
    model=tinyllama_model,
    tokenizer=tinyllama_tokenizer
)
if "STRUCTURE_PRACTICE_QUESTIONS" in agent_1_result:
    library_state["STRUCTURE_PRACTICE_QUESTIONS"] = True

# Agent 2: Discovery mode detection  
agent_2_result = run_agent_with_langchain(
    system_prompt=agent_2_system,
    user_message=user_input,
    model=tinyllama_model,
    tokenizer=tinyllama_tokenizer
)
if "VAUGE_INPUT" in agent_2_result:
    library_state["VAUGE_INPUT"] = True
elif "USER_UNDERSTANDING" in agent_2_result:
    library_state["USER_UNDERSTANDING"] = True

# Agent 3: Follow-up assessment
agent_3_input = f"User Input: {user_input}\nRecent History: {recent_history[-4:]}"
agent_3_result = run_agent_with_langchain(
    system_prompt=agent_3_system,
    user_message=agent_3_input,
    model=tinyllama_model,
    tokenizer=tinyllama_tokenizer
)
# Convert response to boolean
agent_3_boolean = "TRUE" in agent_3_result.upper() or "YES" in agent_3_result.upper()
if agent_3_boolean:
    library_state["PRACTICE_QUESTION_FOLLOWUP"] = True

# Agent 4: Teaching mode assessment
agent_4_input = f"User Input: {user_input}\nRecent History: {recent_history[-4:]}"
agent_4_result = run_agent_with_langchain(
    system_prompt=agent_4_system,
    user_message=agent_4_input,
    model=tinyllama_model,
    tokenizer=tinyllama_tokenizer
)
if "GUIDING_TEACHING" in agent_4_result:
    library_state["GUIDING_TEACHING"] = True
if "STRUCTURE_PRACTICE_QUESTIONS" in agent_4_result:
    library_state["STRUCTURE_PRACTICE_QUESTIONS"] = True
```

### **6. Thinking Agent Processing**
```python
# Determine thinking agent prompt combination
thinking_prompts = []

# Math thinking (always with math topics)
if library_state["LATEX_FORMATTING"]:  # Math detected
    thinking_prompts.append(MATH_THINKING)
    
# Question design (if practice questions needed)
if library_state["STRUCTURE_PRACTICE_QUESTIONS"]:
    thinking_prompts.append(QUESTION_ANSWER_DESIGN)
    
# Reasoning thinking (for teaching/tools/followup)
if (library_state["TOOL_USE_ENHANCEMENT"] or 
    library_state["PRACTICE_QUESTION_FOLLOWUP"] or
    library_state["GUIDING_TEACHING"]):
    thinking_prompts.append(REASONING_THINKING)

# NEW LINE: Join the list into a single string with newlines
thinking_prompts_string = '\n'.join(thinking_prompts)
# You can now print it to see the result or pass it to another function
print(thinking_prompts_string)

# Generate thinking context
thinking_context = thinking_agent.process(
    user_input=user_input,
    conversation_history=recent_history,
    thinking_prompts=thinking_prompts_string
)
```

### **7. Response Prompt Assembly**
```python
# Build prompt_segments from library_state (response agent prompts only)
response_agent_prompts = [
    "VAUGE_INPUT", "USER_UNDERSTANDING", "GENERAL_FORMATTING", 
    "LATEX_FORMATTING", "GUIDING_TEACHING", "STRUCTURE_PRACTICE_QUESTIONS", 
    "PRACTICE_QUESTION_FOLLOWUP", "TOOL_USE_ENHANCEMENT"
]

prompt_segments = [CORE_IDENTITY]  # Always included

for prompt_name in response_agent_prompts:
    if library_state[prompt_name]:
        prompt_segments.append(globals()[prompt_name])  # Get prompt by name

prompt_segments_text = "\n\n".join(prompt_segments)
```

### **8. Final Prompt Construction & Execution**
```python
# Construct complete prompt using template
complete_prompt = f"""
{CORE_IDENTITY}
{prompt_segments_text}
If tools were used, the context and output will be here. Ignore if empty:
Image output: {tool_img_output}
Image context to consider: {tool_context}
Conversation history, if available:        
{recent_history}
Consider any context available to you:
{thinking_context}
Here is the user's current query:
{user_input}
        
"""

# Execute response agent
raw_response = response_agent.invoke(complete_prompt)

# Post-processing pipeline
processed_response = post_processor.process_response(raw_response, user_input)

# Stream to user
stream_response_to_ui(processed_response)

# Reset library_state for next turn
library_state = {key: False for key in library_state.keys()}
```

## **Key Dependencies & Data Flow**

**Global State Provides:**
- `conversation_state` → `recent_history`
- Session persistence across turns

**Library State Tracks:**
- All prompt segment activation states
- Updated by: regex, agents, tool decisions
- Consumed by: prompt assembly logic

**Thinking Agent Context:**
- Input: `user_input`, `recent_history`, active thinking prompts
- Output: `thinking_context` for response agent

**Response Agent Variables:**
- `{prompt_segments}` ← library_state concatenation (response prompts only)
- `{thinking_context}` ← thinking agent output  
- `{recent_history}` ← global state manager
- `{tool_img_output}`, `{tool_context}` ← tool decision engine
- `{user_query}` ← user input

This order ensures proper dependency resolution and maintains clear separation of concerns while leveraging the previously existing global state infrastructure. Many methods, such as post-processing and streaming, are retained.

## Revist
The above portion of this Issue/Solutions documentation is comprised of planning. Some adjustments have been made since the first iteration of this objective. To wrap up this documentation, I am appending charts outlining the architecture of the final implementation and establishing the prompt evaluation methodology for the agentic and response models.

### Architecture


### Evaluation Methodology
As with any AI prompt engineering project, the effectiveness of the prompts will need to be maintained and iterated. Provided that the only model presenting its generated response to the UI is the response model (Phi3-Mimir), I am opting to collect the outputs of all models in log form for review and evaluation, as well as construct evaluation scripts for each that check the outputs. The agents and logic are highly interdependent, where an error in the early stages will have a cascading effect. For this reason, it is vital that the system as a whole, as well as at a unit level, is tested.

