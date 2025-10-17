# Mimir Educational AI Assistant - Architecture Overview
## System Architecture

┌─────────────────────────────────────────────────────────────────┐<br>
│                         Gradio Interface                        │<br>
│                  (Chatbot + Analytics Pages)                    │<br>
└────────────────────┬────────────────────────────────────────────┘<br>
                     │<br>
┌────────────────────▼────────────────────────────────────────────┐<br>
│                    Main Orchestrator (app.py)                   │<br>
│  • Turn management                                              │<br>
│  • Prompt state reset                                           │<br>
│  • Metrics tracking                                             │<br>
└─┬──────────────┬──────────────┬──────────────┬──────────────────┘<br>
  │              │              │              │<br>
  │              │              │              │<br>
┌─▼──────────┐ ┌─▼──────────┐ ┌─▼──────────┐ ┌─▼──────────────────┐<br>
│   Tool     │ │  Routing   │ │ Thinking   │ │    Response        │<br>
│ Decision   │ │  Agents    │ │  Agents    │ │    Agent           │<br>
│  Agent     │ │  (4 agents)│ │ (3 agents) │ │   (Phi-3)          │<br>
└────────────┘ └────────────┘ └────────────┘ └────────────────────┘<br>
     │              │              │              │<br>
     │              │              │              │<br>
┌────▼──────────────▼──────────────▼──────────────▼───────────────┐<br>
│                   Global State Manager                          │<br>
│  • Conversation state                                           │<br>
│  • Prompt state (per-turn)                                      │<br>
│  • Analytics cache                                              │<br>
│  • Evaluation metrics                                           │<br>
│  • SQLite + HF Dataset persistence                              │<br>
└─────────────────────────────────────────────────────────────────┘<br>

## Core Components

### 1. **Agent System** (`agents.py`)
- **ToolDecisionAgent**: Determines visualization needs
- **PromptRoutingAgents**: 4 decision agents for prompt selection
- **ThinkingAgents**: 3 preprocessing agents for complex reasoning
- **ResponseAgent**: Main educational response generation (Phi-3)

### 2. **State Management** (`state_manager.py`)
- **GlobalStateManager**: Thread-safe persistence (SQLite + HF Dataset)
- **PromptStateManager**: Per-turn prompt activation tracking
- **LogicalExpressions**: Regex-based prompt triggers

### 3. **Prompt System** (`prompt_library.py`)
- System prompts for all agents
- Response prompts (dynamically activated)
- Thinking prompts for preprocessing

### 4. **Model Loading Strategy**
- **Build-time**: `preload_from_hub` downloads models during Docker build
- **Runtime**: `compile_model.py` compiles, quantizes, and warms up models
- **Lazy loading**: Agents load models on first GPU call

## Technology Stack

- **Framework**: Gradio (multi-page interface)
- **ML Libraries**: Transformers, BitsAndBytes, Accelerate
- **GPU Management**: ZeroGPU (@spaces.GPU decorators)
- **State Persistence**: SQLite + HuggingFace Datasets
- **Visualization**: Matplotlib (via graph_tool.py)
- **GGUF Support**: llama-cpp-python (optional)

## Key Design Decisions

1. **Lazy Loading**: Models load on first use to minimize startup time
2. **Per-Turn State**: Prompt state resets each turn for clean decision-making
3. **Dual Persistence**: SQLite for fast access, HF Datasets for backup
4. **Agent Separation**: Each agent type has specialized model and prompts
5. **ZeroGPU Integration**: Decorators manage dynamic GPU allocation

# Agent Architecture
## Overview

Mimir uses a multi-agent architecture with 4 distinct agent types, each with specialized responsibilities.

## Agent Hierarchy

┌─────────────────────────────────────────────────────────┐<br>
│                    User Input                           │<br>
└────────────────────┬────────────────────────────────────┘<br>
                     │<br>
            ┌────────▼─────────┐<br>
            │  Tool Decision   │  (Mistral-Small-24B)<br>
            │     Agent        │  - Determines visualization needs<br>
            └────────┬─────────┘<br>
                     │<br>
            ┌────────▼─────────┐<br>
            │  Routing Agents  │  (Mistral-Small-24B, shared)<br>
            │   • Agent 1      │  - Practice questions<br>
            │   • Agent 2      │  - Discovery mode<br>
            │   • Agent 3      │  - Follow-up assessment<br>
            │   • Agent 4      │  - Teaching mode<br>
            └────────┬─────────┘<br>
                     │<br>
            ┌────────▼─────────┐<br>
            │ Thinking Agent   │  (Mistral + GGUF)<br>
            │  • Math          │  - GGUF Mistral (Tree-of-Thought)<br>
            │  • QA Design     │  - Standard Mistral (CoT)<br>
            │  • Reasoning     │  - Standard Mistral (CoT)<br>
            └────────┬─────────┘<br>
                     │<br>
            ┌────────▼─────────┐<br>
            │ Response Agent   │  (Phi-3 Fine-tuned + Base fallback)<br>
            │                  │  - Final response generation<br>
            └──────────────────┘<br>
## Agent Details

1. ToolDecisionAgent

**Purpose**: Determine if visualization tools are needed<br>

**Model**: Mistral-Small-24B (24B parameters)<br>
- 4-bit quantization
- Lazy loading
- ZeroGPU: 60s allocation

**System Prompt**: `TOOL_DECISION`<br>

**Output**: Boolean (True/False)<br>

**Decision Criteria**:
- Mathematical functions or relationships
- Data analysis requirements
- Chart interpretation for practice questions
- Proportional relationships

---

2. PromptRoutingAgents

**Purpose**: Four specialized decision agents for prompt library activation<br>
**Model**: Shared Mistral-Small-24B instance<br>
**Agents**:<br>

#### Agent 1: Practice Questions
- **System Prompt**: `agent_1_system`
- **Output**: Boolean
- **Activates**: `STRUCTURE_PRACTICE_QUESTIONS`

#### Agent 2: Discovery Mode
- **System Prompt**: `agent_2_system`
- **Output**: String ("VAUGE_INPUT" | "USER_UNDERSTANDING" | None)
- **Activates**: `VAUGE_INPUT` or `USER_UNDERSTANDING`

#### Agent 3: Follow-up Assessment
- **System Prompt**: `agent_3_system` (formatted)
- **Output**: Boolean
- **Activates**: `PRACTICE_QUESTION_FOLLOWUP`

#### Agent 4: Teaching Mode
- **System Prompt**: `agent_4_system`
- **Output**: Dict with 2 booleans
- **Activates**: `GUIDING_TEACHING`, `STRUCTURE_PRACTICE_QUESTIONS`

---

### 3. Thinking Agent

**Purpose**: Preprocessing for complex reasoning (outputs context, not final response)

**Models**: 
- Math: GGUF Mistral (llama-cpp-python)
- QA Design & Reasoning: Standard Mistral-Small-24B

**Agents**:

#### Math Thinking
- **System Prompt**: `MATH_THINKING`
- **Method**: Tree-of-Thought reasoning
- **Output**: Mathematical context with LaTeX
- **Max Tokens**: 512

#### Question/Answer Design
- **System Prompt**: `QUESTION_ANSWER_DESIGN` (formatted)
- **Method**: Chain-of-Thought
- **Output**: Question design context
- **Max Tokens**: 512

#### Reasoning Thinking
- **System Prompt**: `REASONING_THINKING`
- **Method**: Chain-of-Thought
- **Output**: General reasoning context
- **Max Tokens**: 512

---

### 4. ResponseAgent
<br>
**Purpose**: Final educational response generation<br>

**Model**: Phi-3-mini-4k-instruct<br>
- **Primary**: Fine-tuned (`jdesiree/Mimir-Phi-3.5`)
- **Fallback**: Base Microsoft model
- 4-bit quantization
- ZeroGPU: 120s allocation
- Dynamic system prompt construction

**System Prompt**: `CORE_IDENTITY` (always) + dynamic prompts from library

**Features**:
- PEFT-enabled
- Accelerate integration
- Automatic fallback on fine-tuned model failure
- Post-processing pipeline

---

## Model Loading Strategy

### Build Time (preload_from_hub)
```yaml
preload_from_hub:
  - jdesiree/Mimir-Phi-3.5              # Full repo
  - microsoft/Phi-3-mini-4k-instruct    # Full repo
  - yentinglin/Mistral-Small-24B...     # Full repo
  - brittlewis12/...GGUF mistral...gguf # Specific file
  - thenlper/gte-small                  # Full repo
```

### Runtime (compile_model.py)
1. Check for cached models
2. Load from HF cache (already downloaded)
3. Apply quantization
4. Run warmup inference
5. Create markers for the fast path

### First Agent Call (Lazy Loading)
1. Check if the model is already loaded
2. If not, load from cache
3. Apply configurations
4. Ready for inference

---

## Message Format
<br>
All agents use the LangChain message format:<br>

```python
from langchain_core.messages import SystemMessage, HumanMessage

messages = [
    SystemMessage(content=system_prompt),
    HumanMessage(content=user_message)
]
```

## ZeroGPU Integration
<br>
Each agent is decorated with:<br>

```python
@spaces.GPU(duration=60)  # Duration in seconds
def agent_method(self, ...):
    # GPU allocated dynamically for this call
    ...
```

## Agent Communication Flow

```
User Input
    ↓
ToolDecisionAgent.should_use_visualization()
    ↓
PromptRoutingAgents.agent_1_practice_questions()
PromptRoutingAgents.agent_2_discovery_mode()
PromptRoutingAgents.agent_3_followup_assessment()
PromptRoutingAgents.agent_4_teaching_mode()
    ↓ (prompt states updated)
ThinkingAgents.process() [if needed]
    ↓ (thinking context generated)
ResponseAgent.invoke(complete_prompt)
    ↓
Final Response (post-processed)
```
```

## 3. **STATE_MANAGEMENT.md**

```markdown
# State Management Architecture

## Overview

Mimir uses a dual-layer state management system:
1. **Global State**: Persistent across sessions (SQLite + HF Dataset)
2. **Prompt State**: Per-turn, resets each interaction

## Architecture

```
┌────────────────────────────────────────────────────────┐<br>
│           GlobalStateManager                           │<br>
│  ┌──────────────────────────────────────────────────┐  │<br>
│  │  PromptStateManager (per-turn)                   │  │<br>
│  │  • Resets at the start of each turn              │  │<br>
│  │  • Tracks active prompts                         │  │<br>
│  └──────────────────────────────────────────────────┘  │<br>
│                                                        │<br>
│  Persistent State:                                     │<br>
│  • Conversation state (chat history)                   │<br>
│  • Analytics cache                                     │<br>
│  • ML model cache                                      │<br>
│  • Evaluation metrics                                  │<br>
│                                                        │<br>
│  Persistence Layer:                                    │<br>
│  • SQLite (fast local access)                          │<br>
│  • HF Dataset (backup & sync)                          │<br>
└────────────────────────────────────────────────────────┘<br>

## Prompt State Manager

### Purpose
Manages which prompt segments are active for the current turn only.<br>

### State Dictionary
```python
{
    # Thinking prompts
    "MATH_THINKING": False,
    "QUESTION_ANSWER_DESIGN": False,
    "REASONING_THINKING": False,
    
    # Response prompts
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

### Lifecycle
```
Turn Start
    ↓
global_state_manager.reset_prompt_state()  # All False
    ↓
Agents make decisions
    ↓
prompt_state.update("PROMPT_NAME", True)  # Activate prompts
    ↓
Prompt assembly uses get_active_response_prompts()
    ↓
Response generated
    ↓
[Next Turn - State resets again]
```

### Key Methods

# Get manager
`prompt_state = global_state_manager.get_prompt_state_manager()`

# Reset for new turn
`prompt_state.reset()`

# Update state
`prompt_state.update("LATEX_FORMATTING", True)`
`prompt_state.update_multiple({"GUIDING_TEACHING": True, "GENERAL_FORMATTING": True})`

# Query state
`is_active = prompt_state.is_active("MATH_THINKING")`
`active_prompts = prompt_state.get_active_prompts()`
`response_prompts = prompt_state.get_active_response_prompts()`
`thinking_prompts = prompt_state.get_active_thinking_prompts()`

---

## Global State Manager

### Purpose
Thread-safe persistence for all application state across sessions.<br>

### State Categories

#### 1. Conversation State
```python
{
    'chat_history': [  # For Gradio display
        {"role": "user", "content": "..."},
        {"role": "assistant", "content": "..."}
    ],
    'conversation_state': [  # For agent context
        {"role": "user", "content": "..."},
        {"role": "assistant", "content": "..."}
    ],
    'last_accessed': datetime,
    'created': datetime
}
```

#### 2. Analytics State
```python
{
    'project_stats': {...},
    'recent_interactions': [...],
    'dashboard_html': "...",
    'last_refresh': datetime,
    'export_history': [...]
}
```

#### 3. Evaluation State
```python
{
    'educational_quality_scores': [...],
    'rag_performance_metrics': [...],
    'prompt_classification_accuracy': [...],
    'user_feedback_history': [...],
    'aggregate_metrics': {...}
}
```

#### 4. ML Model Cache
```python
{
    'model_type': {
        'model': model_object,
        'cached_at': datetime,
        'metadata': {...},
        'access_count': int
    }
}
```

### Persistence Strategy

#### SQLite (Primary)
- **Location**: `mimir_analytics.db`
- **Purpose**: Fast local access
- **Tables**: `conversations`, `analytics`, `evaluations`, `classifications`

#### HuggingFace Dataset (Backup)
- **Repo**: `jdesiree/mimir_analytics`
- **Purpose**: Cloud backup & sync
- **Sync Interval**: Every 3600 seconds (1 hour)

### Thread Safety
All state operations protected by `threading.Lock()`:
```python
with self._lock:
    # Safe state modification
    self._states[session_id] = new_state
```

### Cleanup Strategy
- **Interval**: Every 3600 seconds
- **Max Age**: 24 hours
- **Action**: Remove old unused states to prevent memory leaks

---

## Logical Expressions

### Purpose
Regex-based triggers for automatic prompt activation.

### Current Rules
```python
# Math keywords → LATEX_FORMATTING
math_regex = r'\b(math|calculus|algebra|geometry|equation|formula|solve|calculate|derivative|integral|trigonometry|statistics|probability)\b'

# Always active
GENERAL_FORMATTING: Always True
```

### Usage
```python
logical_expressions = LogicalExpressions()
logical_expressions.apply_all_checks(user_input, prompt_state)
```

---

## State Flow Example
```
# Turn 1: Start
  global_state_manager.reset_prompt_state()  
  prompt_state = global_state_manager.get_prompt_state_manager()  
**All prompts: False**

# Agent decisions
`tool_agent.should_use_visualization(user_input)  # Returns True`
`prompt_state.update("TOOL_USE_ENHANCEMENT", True)`

`routing_agents.agent_1_practice_questions(...)  # Returns True`
`prompt_state.update("STRUCTURE_PRACTICE_QUESTIONS", True)`

`logical_expressions.apply_all_checks(user_input, prompt_state)`

# Detects "calculus" → `prompt_state.update("LATEX_FORMATTING", True)`

# Prompt assembly
active_prompts = prompt_state.get_active_response_prompts()
# Returns: ["GENERAL_FORMATTING", "LATEX_FORMATTING", 
#           "STRUCTURE_PRACTICE_QUESTIONS", "TOOL_USE_ENHANCEMENT"]

# Response generation
response = response_agent.invoke(complete_prompt)

# Save to global state
global_state_manager.update_conversation_state(
    chat_history, conversation_state
)

# Turn 2: Reset
global_state_manager.reset_prompt_state()
# All prompts: False again (clean slate)
```

---

## Database Schema

### conversations
```sql
CREATE TABLE conversations (
    session_id TEXT PRIMARY KEY,
    chat_history TEXT,           -- JSON
    conversation_state TEXT,      -- JSON
    last_accessed TEXT,           -- ISO datetime
    created TEXT                  -- ISO datetime
)
```

### analytics
```sql
CREATE TABLE analytics (
    session_id TEXT PRIMARY KEY,
    project_stats TEXT,          -- JSON
    recent_interactions TEXT,     -- JSON
    dashboard_html TEXT,
    last_refresh TEXT,           -- ISO datetime
    export_history TEXT          -- JSON
)
```

### evaluations
```sql
CREATE TABLE evaluations (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT,
    timestamp TEXT,
    metric_type TEXT,
    metric_data TEXT             -- JSON
)
```

---

## Key Design Decisions

1. **Per-Turn Reset**: Prompt state resets each turn for clean decision-making
2. **Dual Persistence**: SQLite for speed, HF Dataset for backup
3. **Thread Safety**: All state operations locked
4. **Lazy Cleanup**: Periodic cleanup prevents memory leaks
5. **Session-based**: Multiple concurrent users supported
```
## 4. **DATA_FLOW.md**

```markdown
# Data Flow Architecture

## Complete Turn Flow

```
┌─────────────────────────────────────────────────────────────────┐<br>
│                         USER INPUT                              │<br>
└────────────────────┬────────────────────────────────────────────┘<br>
                     │<br>
                     ▼<br>
┌─────────────────────────────────────────────────────────────────┐<br>
│ STEP 1: RESET PROMPT STATE                                      │<br>
│ global_state_manager.reset_prompt_state()                       │<br>
│ All prompt flags → False                                        │<br>
└────────────────────┬────────────────────────────────────────────┘<br>
                     │<br>
                     ▼<br>
┌─────────────────────────────────────────────────────────────────┐<br>
│ STEP 2: PROCESS USER INPUT                                      │<br>
│ • Get conversation history (last 8 messages)                    │<br>
│ • Format for agents                                             │<br>
└────────────────────┬────────────────────────────────────────────┘<br>
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ STEP 3: TOOL DECISION                                            │
│ tool_decision_result = tool_agent.should_use_visualization()     │
│   ↓                                                              │
│ if True: prompt_state.update("TOOL_USE_ENHANCEMENT", True)       │
└────────────────────┬────────────────────────────────────────────┘
                     │<br>
                     ▼<br>
┌─────────────────────────────────────────────────────────────────┐<br>
│ STEP 4: REGEX LOGICAL EXPRESSIONS                               │<br>
│ logical_expressions.apply_all_checks(user_input, prompt_state)  │<br>
│   ↓                                                             │<br>
│ • GENERAL_FORMATTING → Always True                              │<br>
│ • Math keywords detected → LATEX_FORMATTING = True              │<br>
└────────────────────┬────────────────────────────────────────────┘<br>
                     │<br>
                     ▼<br>
┌─────────────────────────────────────────────────────────────────┐<br>
│ STEP 5: SEQUENTIAL AGENT EXECUTION                              │<br>
│                                                                 │<br>
│ Agent 1: Practice Questions                                     │<br>
│   result = routing_agents.agent_1_practice_questions()          │<br>
│   if True: prompt_state.update("STRUCTURE_PRACTICE_QUESTIONS")  │<br>
│                                                                 │<br>
│ Agent 2: Discovery Mode                                         │<br>
│   result = routing_agents.agent_2_discovery_mode()              │<br>
│   if result: prompt_state.update(result, True)                  │<br>
│                                                                 │<br>
│ Agent 3: Follow-up Assessment                                   │<br>
│   result = routing_agents.agent_3_followup_assessment()         │<br>
│   if True: prompt_state.update("PRACTICE_QUESTION_FOLLOWUP")    │<br>
│                                                                 │<br>
│ Agent 4: Teaching Mode                                          │<br>
│   results = routing_agents.agent_4_teaching_mode()              │<br>
│   prompt_state.update_multiple(results)                         │<br>
└────────────────────┬────────────────────────────────────────────┘<br>
                     │<br>
                     ▼<br>
┌─────────────────────────────────────────────────────────────────┐<br>
│ STEP 6: THINKING AGENT PROCESSING                               │<br>
│                                                                 │<br>
│ Determine active thinking agents:                               │<br>
│ • LATEX_FORMATTING active? → MATH_THINKING                      │<br>
│ • STRUCTURE_PRACTICE_QUESTIONS? → QUESTION_ANSWER_DESIGN        │<br>
│ • TOOL/FOLLOWUP/TEACHING active? → REASONING_THINKING           │<br>
│                                                                 │<br>
│ Execute thinking agents:                                        │<br>
│   thinking_context = thinking_agents.process(                   │<br>
│       user_input, history, thinking_prompts,                    │<br>
│       tool_img_output, tool_context                             │<br>
│   )                                                             │<br>
└────────────────────┬────────────────────────────────────────────┘<br>
                     │<br>
                     ▼<br>
┌─────────────────────────────────────────────────────────────────┐<br>
│ STEP 7: RESPONSE PROMPT ASSEMBLY                                │<br>
│                                                                 │<br>
│ response_prompts = prompt_state.get_active_response_prompts()   │<br>
│                                                                 │<br>
│ Build prompt segments:                                          │<br>
│ • CORE_IDENTITY (always)                                        │<br>
│ • For each active prompt: add from prompt_map                   │<br>
│                                                                 │<br>
│ prompt_segments_text = "\n\n".join(prompt_segments)             │<br>
└────────────────────┬────────────────────────────────────────────┘<br>
                     │<br>
                     ▼<br>
┌─────────────────────────────────────────────────────────────────┐<br>
│ STEP 8: FINAL PROMPT CONSTRUCTION                               │<br>
│                                                                 │<br>
│ complete_prompt = f"""                                          │<br>
│ {prompt_segments_text}                                          │<br>
│                                                                 │<br>
│ Tool context: {tool_img_output} {tool_context}                  │<br>
│ History: {recent_history_formatted}                             │<br>
│ Thinking context: {thinking_context}                            │<br>
│ User query: {user_input}                                        │<br>
│ Knowledge cutoff: {CURRENT_YEAR}                                │<br>
│ """                                                             │<br>
└────────────────────┬────────────────────────────────────────────┘<br>
                     │<br>
                     ▼<br>
┌─────────────────────────────────────────────────────────────────┐<br>
│ STEP 9: RESPONSE GENERATION                                     │<br>
│ raw_response = response_agent.invoke(complete_prompt)           │<br>
│   ↓                                                             │<br>
│ Uses Phi-3 (fine-tuned or base fallback)                        │<br>
│ Max tokens: 350                                                 │<br>
│ Temperature: 0.7                                                │<br>
└────────────────────┬────────────────────────────────────────────┘<br>
                     │<br>
                     ▼<br>
┌─────────────────────────────────────────────────────────────────┐<br>
│ STEP 10: POST-PROCESSING                                        │<br>
│ processed_response = post_processor.process_response()          │<br>
│   ↓                                                             │<br>
│ • Clean artifacts (<|end|>, ###, etc.)                          │<br>
│ • Intelligent truncation                                        │<br>
│ • Enhance readability                                           │<br>
│ • Quality validation                                            │<br>
└────────────────────┬────────────────────────────────────────────┘<br>
                     │<br>
                     ▼<br>
┌─────────────────────────────────────────────────────────────────┐<br>
│ STEP 11: METRICS TRACKING                                       │<br>
│                                                                 │<br>
│ • Educational quality evaluation                                │<br>
│ • Response quality scoring                                      │<br>
│ • Log to SQLite database                                        │<br>
│ • Update global state                                           │<br>
│                                                                 │<br>
│ metrics = {                                                     │<br>
│     "response_time": duration,                                  │<br>
│     "quality_score": score,                                     │<br>
│     "educational_score": ed_score,                              │<br>
│     "prompt_mode": active_prompts,                              │<br>
│     "tools_used": bool,                                         │<br>
│     "thinking_agents": list                                     │<br>
│ }                                                               │<br>
└────────────────────┬────────────────────────────────────────────┘<br>
                     │<br>
                     ▼<br>
┌─────────────────────────────────────────────────────────────────┐<br>
│                    FINAL RESPONSE TO USER                       │<br>
└─────────────────────────────────────────────────────────────────┘<br>
```

---

## Detailed Step Breakdowns

### Step 3: Tool Decision Flow

```
user_input: "Can you graph the function f(x) = x^2?"
    ↓
tool_agent.should_use_visualization(user_input)
    ↓
[GPU allocated - 60s]
    ↓
Format with TOOL_DECISION system prompt
    ↓
Mistral inference (max 10 tokens)
    ↓
Parse output: "YES" or "NO"
    ↓
Result: True (contains "graph")
    ↓
prompt_state.update("TOOL_USE_ENHANCEMENT", True)
```

### Step 5: Routing Agent Cascade

```
Agent 1 (Practice Questions)<br>
    Input: user_input + recent_history (last 4 msgs)<br>
    System: agent_1_system<br>
    Output: Boolean<br>
    GPU: 50s<br>
    Decision: False (user wants graph, not practice)<br>
    ↓<br>
Agent 2 (Discovery Mode)<br>
    Input: user_input only<br>
    System: agent_2_system<br>
    Output: "VAUGE_INPUT" | "USER_UNDERSTANDING" | None<br>
    GPU: 50s<br>
    Decision: None (clear request for graph)<br>
    ↓<br>
Agent 3 (Follow-up Assessment)<br>
    Input: user_input + recent_history (last 4 msgs)<br>
    System: agent_3_system (formatted with STRUCTURE_PRACTICE_QUESTIONS)<br>
    Output: Boolean<br>
    GPU: 50s<br>
    Decision: False (not a follow-up)<br>
    ↓<br>
Agent 4 (Teaching Mode)<br>
    Input: user_input + recent_history (last 4 msgs)<br>
    System: agent_4_system<br>
    Output: Dict{"GUIDING_TEACHING": bool, "STRUCTURE_PRACTICE_QUESTIONS": bool}<br>
    GPU: 50s<br>
    Decision: {"GUIDING_TEACHING": True, "STRUCTURE_PRACTICE_QUESTIONS": False}<br>
    ↓<br>
prompt_state after routing:<br>

```
{
    "GENERAL_FORMATTING": True,
    "TOOL_USE_ENHANCEMENT": True,
    "GUIDING_TEACHING": True
}
```

### Step 6: Thinking Agent Activation

```
Check prompt_state for thinking triggers:
    ↓
LATEX_FORMATTING active? → No
    Skip MATH_THINKING
    ↓
STRUCTURE_PRACTICE_QUESTIONS active? → No
    Skip QUESTION_ANSWER_DESIGN
    ↓
TOOL_USE_ENHANCEMENT active? → Yes
    Activate REASONING_THINKING
    ↓
thinking_prompts_list = ["REASONING_THINKING"]
prompt_state.update("REASONING_THINKING", True)
    ↓
thinking_agents.process(
    user_input, history, "REASONING_THINKING",
    tool_img_output, tool_context
)
    ↓
[GPU allocated - 60s]
    ↓
reasoning_thinking() called:
    System: REASONING_THINKING
    Output: Context with:
        - User Knowledge Summary
        - User Understanding analysis
        - Previous Actions review
        - Reference Fact Sheet
    Max tokens: 512
    ↓
thinking_context = "=== Reasoning Context ===\n{output}"
```

### Step 7-8: Prompt Assembly

```
active_response_prompts = ["GENERAL_FORMATTING", "TOOL_USE_ENHANCEMENT", "GUIDING_TEACHING"]
    ↓
Build segments:
    1. CORE_IDENTITY (always)
    2. GENERAL_FORMATTING
    3. TOOL_USE_ENHANCEMENT
    4. GUIDING_TEACHING
    ↓
prompt_segments_text = """
You are Mimir, an expert multi-concept tutor...

## General Formatting Guidelines
- Headings must be on their own line...

## Tool Usage for Educational Enhancement
Apply when teaching concepts that benefit...

As a skilled educator, considering the conversation...
"""
    ↓
complete_prompt = """
{prompt_segments_text}

Tool context: [empty if no tool output]

History: user: "Can you graph the function f(x) = x^2?"

Thinking context: 
=== Reasoning Context ===
User wants to visualize a quadratic function...

Current query: Can you graph the function f(x) = x^2?

The current year is 2025...
"""
```

### Step 9: Response Generation

```
complete_prompt
    ↓
response_agent.invoke(complete_prompt)
    ↓
[GPU allocated - 120s]
    ↓
Check if model loaded:
    if not: _load_and_prepare_model()
        ↓ Try fine-tuned model first
        ↓ Fallback to base if fails
    ↓
Format using Phi-3 chat template:
    messages = [{"role": "user", "content": complete_prompt}]
    formatted = tokenizer.apply_chat_template(messages)
    ↓
Tokenize (max 3500 input tokens, truncate if needed)
    ↓
Generate:
    max_new_tokens: 350
    temperature: 0.7
    repetition_penalty: 1.15
    ↓
Decode new tokens only
    ↓
Strip stop words: "User:", "<|end|>", "<|assistant|>"
    ↓
raw_response = "I'd be happy to help you visualize the quadratic function..."
```

---

## Message Formats Between Components

### Agent → Prompt State
```python
# Simple boolean
prompt_state.update("LATEX_FORMATTING", True)

# String value (Agent 2)
result = "VAUGE_INPUT"
prompt_state.update(result, True)

# Dictionary (Agent 4)
results = {"GUIDING_TEACHING": True, "STRUCTURE_PRACTICE_QUESTIONS": False}
prompt_state.update_multiple(results)
```

### Thinking Agents → Response Agent
```python
thinking_context = """
=== Mathematical Thinking Context ===
The problem involves solving a quadratic equation...

=== Reasoning Context ===
User Understanding: Beginner level algebra...
"""
# Inserted into complete_prompt
```

### Response Agent → Post-Processor
```python
raw_response = "I'd be happy to help! <|end|> ###"
    ↓
processed = post_processor.process_response(raw_response, user_input)
    ↓
cleaned_response = "I'd be happy to help!"
```

### Post-Processor → Gradio (Streaming)
```python
for chunk in post_processor.process_and_stream_response(raw, query):
    yield chunk  # "I'd", "I'd be", "I'd be happy", ...
```

---

## State Persistence Flow

### During Turn
```
User input received
    ↓
Load current state:
    conversation_state = global_state_manager.get_conversation_state()
    chat_history = conversation_state['chat_history']
    ↓
Process turn...
    ↓
Update state:
    conversation_state.append({"role": "user", "content": input})
    conversation_state.append({"role": "assistant", "content": response})
    chat_history.append(...)
    ↓
Save to global state:
    global_state_manager.update_conversation_state(
        chat_history, conversation_state, session_id
    )
    ↓
Triggers:
    - SQLite update (immediate)
    - HF Dataset backup (if >1hr since last)
```

### Metrics Tracking
```
After response generated:
    ↓
Calculate metrics:
    quality_metrics = evaluate_educational_quality_with_tracking(...)
    quality_score = calculate_response_quality(response)
    ↓
Build metrics dict:
    metrics = {
        "conversation_start": timestamp,
        "response_time": duration,
        "quality_score": float,
        "educational_score": float,
        "prompt_mode": "GENERAL,TOOL,GUIDING",
        "tools_used": 1,
        "thinking_agents": "REASONING_THINKING",
        "active_adapter": "fine-tuned"
    }
    ↓
Log to database:
    log_metrics_to_database("Mimir", run_id, metrics)
    ↓
Update global state:
    global_state_manager.add_educational_quality_score(...)
```

---

## Error Handling Flow

### Agent Failure
```
try:
    result = agent.invoke(...)
except Exception as e:
    logger.error(f"Agent failed: {e}")
    result = fallback_value  # Continue pipeline
```

### Model Loading Failure
```
try:
    model = load_fine_tuned_model()
    model_type = "fine-tuned"
except Exception:
    model = load_base_model()
    model_type = "base-fallback"
    logger.warning("Using base model fallback")
```

### Response Generation Failure
```
try:
    response = response_agent.invoke(prompt)
except Exception as e:
    logger.error(f"Generation failed: {e}")
    response = "I encountered an error: {e}"
finally:
    # Always save state
    global_state_manager.update_conversation_state(...)
```

---

## Timing Breakdown (Typical Turn)

```
Total Turn Time: ~25-40 seconds

Step 1: Reset prompt state         < 0.01s
Step 2: Process input               < 0.1s
Step 3: Tool decision              ~3-5s (GPU inference)
Step 4: Regex checks               < 0.01s
Step 5: Routing agents (4x)        ~12-20s (4 GPU calls)
Step 6: Thinking agents            ~4-8s (if activated)
Step 7: Prompt assembly            < 0.1s
Step 8: Final construction         < 0.1s
Step 9: Response generation        ~5-10s (GPU inference)
Step 10: Post-processing           ~0.5-1s (streaming)
Step 11: Metrics tracking          < 0.5s
```

---

## Gradio Callback Flow

```
User types message → clicks submit
    ↓
add_user_message(message, chat_history, conversation_state)
    ↓
    - Append to conversation_state
    - Append to chat_history
    - Update global_state_manager
    ↓
add_loading_animation(chat_history, conversation_state)
    ↓
    - Add GIF to chat_history
    - Update display
    ↓
generate_response(chat_history, conversation_state)
    ↓
    - Get last user message
    - Call orchestrate_turn(message)
    - Remove loading animation
    - Stream response word-by-word
    - Update chat_history
    - Update conversation_state
    - Yield updates (Gradio streaming)
    ↓
Display final response
```
```

## 5. **PROMPT_SYSTEM.md**

```markdown
# Prompt System Architecture

## Overview

Mimir uses a hierarchical prompt system with:
- **System prompts** for agents (fixed per agent type)
- **Response prompts** for ResponseAgent (dynamically activated)
- **Thinking prompts** for preprocessing agents

All prompts stored in `prompt_library.py`.

---

## Prompt Categories

### 1. System Prompts (Agent Instructions)

#### CORE_IDENTITY
**Type**: Response Agent base prompt (always included)

**Purpose**: Establishes Mimir's educational persona

**Key Elements**:
- Expert multi-concept tutor identity
- Communication standards (friendly, professional, no vulgar language)
- Follow-up response guidelines
- Grading practice question answers

**Always Active**: Yes

---

#### TOOL_DECISION
**Type**: ToolDecisionAgent system prompt

**Purpose**: Determines if visualization needed

**Output Format**: "YES" or "NO"

**Decision Criteria**:
- Mathematical functions or trends
- Statistical concepts
- Practice questions requiring charts
- Proportional relationships

---

#### agent_1_system
**Type**: PromptRoutingAgent #1 system prompt

**Purpose**: Determine if practice questions needed

**Output Format**: 
- "STRUCTURE_PRACTICE_QUESTIONS" (if needed)
- "No Practice questions are needed." (if not)

**Triggers**:
- User explicitly requests practice questions
- User wants to gauge understanding
- Previous instruction + user shows understanding

**Excludes**:
- Conversational/nonsense input
- Straightforward questions needing direct answers

---

#### agent_2_system
**Type**: PromptRoutingAgent #2 system prompt

**Purpose**: Detect vague input or unclear needs

**Output Format**:
- "VAUGE_INPUT USER_UNDERSTANDING"
- "USER_UNDERSTANDING"
- "VAUGE_INPUT"
- "Neither is applicable."

**Classifications**:
- **VAUGE_INPUT**: Overly vague, uninterpretable, simple greeting
- **USER_UNDERSTANDING**: Conversational greeting, topic change, needs discovery
- **Neither**: Coherent message, direct request, or follow-up

---

#### agent_3_system
**Type**: PromptRoutingAgent #3 system prompt

**Purpose**: Determine if current input is practice question follow-up

**Output Format**:
- "PRACTICE_QUESTION_FOLLOWUP True" (if follow-up)
- "Not a followup" (if not)

**Requires**: Formatted with `{STRUCTURE_PRACTICE_QUESTIONS}` for context

---

#### agent_4_system
**Type**: PromptRoutingAgent #4 system prompt

**Purpose**: Assess teaching mode and practice structure needs

**Output Format**:
- "GUIDING_TEACHING"
- "STRUCTURE_PRACTICE_QUESTIONS"
- "GUIDING_TEACHING STRUCTURE_PRACTICE_QUESTIONS"
- "Neither Apply"

**Triggers**:
- **GUIDING_TEACHING**: User requesting information, confused, continuing discussion
- **STRUCTURE_PRACTICE_QUESTIONS**: Positive response to instruction, direct request

---

### 2. Response Prompts (Dynamically Activated)

#### GENERAL_FORMATTING
**Activation**: Always (via LogicalExpressions)

**Content**:
- Heading guidelines (##, ###)
- Paragraph separation
- Structure for complex vs simple responses
- No emojis rule

---

#### LATEX_FORMATTING
**Activation**: Math keywords detected (via LogicalExpressions)

**Content**:
- Inline math: `$ ... $`
- Display math: `$$ ... $$`
- Literal dollar signs: `\$`
- Literal parentheses: `\(` and `\)`

**Triggers**: Regex match for math keywords (calculus, algebra, equation, etc.)

---

#### VAUGE_INPUT
**Activation**: Agent 2 returns "VAUGE_INPUT"

**Content**:
- Use discovery tactics
- Ask how to help
- Suggest creating practice questions or exploring topics

---

#### USER_UNDERSTANDING
**Activation**: Agent 2 returns "USER_UNDERSTANDING"

**Content**:
- Uncover user's current knowledge
- Approach instructing to facilitate learning
- Ask targeted questions about concepts

---

#### GUIDING_TEACHING
**Activation**: Agent 4 returns "GUIDING_TEACHING"

**Content**:
- Guide understanding (don't give full solutions)
- Academic integrity guidelines:
  - Break into conceptual components
  - Ask clarifying questions
  - Provide analogous examples
  - Encourage original thinking
- Subject-specific approaches (Math, Multiple-choice, Essays, Factual)

---

#### STRUCTURE_PRACTICE_QUESTIONS
**Activation**: Agent 1 or Agent 4 returns True

**Content**:
- Question formatting guidelines (1-2 questions)
- Integration with tool output
- Data reference formatting (tables, images)
- Answer option formats:
  - Single option multiple choice (A, B, C, D)
  - All that apply (checkboxes)
  - Written response

---

#### PRACTICE_QUESTION_FOLLOWUP
**Activation**: Agent 3 returns True

**Content**:
- Assess questions from previous turn
- Identify correct answers
- Grade user response
- Provide feedback:
  - If correct: Congratulate, offer to continue
  - If incorrect: Constructive feedback + rationale
  - If no answer: Assess input, offer help

---

#### TOOL_USE_ENHANCEMENT
**Activation**: ToolDecisionAgent returns True

**Content**:
- Tool usage criteria (teaching functions, stats, charts)
- `Create_Graph_Tool` signature and parameters
- Example scenarios
- Educational context guidance

---

### 3. Thinking Prompts (Preprocessing Agents)

#### MATH_THINKING
**Type**: Tree-of-Thought prompting

**Model**: GGUF Mistral (llama-cpp-python)

**Purpose**: Math-specific reasoning preprocessing

**Output Structure**:
```
**Key Terms**
- Definition 1
- Definition 2

**Principle: [Name]**
Description

**Formula**
LaTeX math

**Step-by-Step Solution** (if complex)
1. Step with rationale
2. Step with rationale

**Summary** (if complex)
- Bullet points
```

**Decision Tree**:
- 1A: Low-level math (< 5 steps) → Minimal detail
- 1B: Complex math (> 5 steps) → Detailed with numbered steps

---

#### QUESTION_ANSWER_DESIGN
**Type**: Chain-of-Thought prompting

**Model**: Standard Mistral-Small-24B

**Purpose**: Practice question formulation preprocessing

**Requires Formatting**:
- `{tool_img_output}`
- `{tool_context}`
- `{STRUCTURE_PRACTICE_QUESTIONS}`
- `{LATEX_FORMATTING}`

**Output Structure**:
```
[Topic/Domain Summary]

Practice Question
[Question text]
[Data table or image reference if needed]

[Answer bank with correct answers labeled]
```

**Process**:
1. Assess topic and conversation history
2. Produce practice question (not from history examples)
3. Create answer bank or written response examples

---

#### REASONING_THINKING
**Type**: Chain-of-Thought prompting

**Model**: Standard Mistral-Small-24B

**Purpose**: General reasoning preprocessing

**Output Structure**:
```
### User Knowledge Summary
[What user is asking + conversation summary]

### User Understanding
[User's grasp + potential concepts to cover]

### Previous Actions
[Model's previous steps + effectiveness]

### Reference Fact Sheet
- **Topic 1**
    - Atomic fact 1
    - Atomic fact 2
- **Topic 2**
    - Atomic fact 1
```

---

## Prompt Selection Logic

### Activation Matrix

| Prompt Name | Activated By | Condition |
|------------|--------------|-----------|
| CORE_IDENTITY | Always | N/A |
| GENERAL_FORMATTING | LogicalExpressions | Always True |
| LATEX_FORMATTING | LogicalExpressions | Math keywords detected |
| VAUGE_INPUT | Agent 2 | Returns "VAUGE_INPUT" |
| USER_UNDERSTANDING | Agent 2 | Returns "USER_UNDERSTANDING" |
| GUIDING_TEACHING | Agent 4 | Returns in dict |
| STRUCTURE_PRACTICE_QUESTIONS | Agent 1 or Agent 4 | Returns True |
| PRACTICE_QUESTION_FOLLOWUP | Agent 3 | Returns True |
| TOOL_USE_ENHANCEMENT | ToolDecisionAgent | Returns True |
| MATH_THINKING | Step 6 logic | LATEX_FORMATTING active |
| QUESTION_ANSWER_DESIGN | Step 6 logic | STRUCTURE_PRACTICE_QUESTIONS active |
| REASONING_THINKING | Step 6 logic | TOOL/FOLLOWUP/TEACHING active |

---

## Prompt Assembly Process

### Step 1: Get Active Response Prompts
```python
response_prompt_names = prompt_state.get_active_response_prompts()
# Returns list like: ["GENERAL_FORMATTING", "LATEX_FORMATTING", "GUIDING_TEACHING"]
```

### Step 2: Build Segments
```python
prompt_segments = [CORE_IDENTITY]  # Always first

prompt_map = {
    "VAUGE_INPUT": VAUGE_INPUT,
    "USER_UNDERSTANDING": USER_UNDERSTANDING,
    "GENERAL_FORMATTING": GENERAL_FORMATTING,
    "LATEX_FORMATTING": LATEX_FORMATTING,
    "GUIDING_TEACHING": GUIDING_TEACHING,
    "STRUCTURE_PRACTICE_QUESTIONS": STRUCTURE_PRACTICE_QUESTIONS,
    "PRACTICE_QUESTION_FOLLOWUP": PRACTICE_QUESTION_FOLLOWUP,
    "TOOL_USE_ENHANCEMENT": TOOL_USE_ENHANCEMENT,
}

for prompt_name in response_prompt_names:
    if prompt_name in prompt_map:
        prompt_segments.append(prompt_map[prompt_name])
```

### Step 3: Join Segments
```python
prompt_segments_text = "\n\n".join(prompt_segments)
```

### Step 4: Add Context
```python
complete_prompt = f"""
{prompt_segments_text}

Tool context: {tool_img_output} {tool_context}
History: {recent_history_formatted}
Thinking context: {thinking_context}
User query: {user_input}
Knowledge cutoff: {CURRENT_YEAR}
"""
```

---

## Example Prompt Assemblies

### Example 1: Simple Math Question

**User Input**: "What is the derivative of x^2?"

**Active Prompts**:
- CORE_IDENTITY (always)
- GENERAL_FORMATTING (always)
- LATEX_FORMATTING (math keywords)
- GUIDING_TEACHING (Agent 4)

**Thinking Agents**: MATH_THINKING

**Final Prompt Structure**:
```
[CORE_IDENTITY text]

[GENERAL_FORMATTING text]

[LATEX_FORMATTING text]

[GUIDING_TEACHING text]

Tool context: [empty]
History: [empty or previous conversation]

Thinking context:
=== Mathematical Thinking Context ===
**Key Terms**
- Derivative: Rate of change of a function
**Principle: Power Rule**
For f(x) = x^n, f'(x) = nx^(n-1)

User query: What is the derivative of x^2?

The current year is 2025...
```

---

### Example 2: Practice Question Request

**User Input**: "Can you give me practice questions on the Civil War?"

**Active Prompts**:
- CORE_IDENTITY
- GENERAL_FORMATTING
- STRUCTURE_PRACTICE_QUESTIONS (Agent 1)

**Thinking Agents**: QUESTION_ANSWER_DESIGN

**Final Prompt Structure**:
```
[CORE_IDENTITY text]

[GENERAL_FORMATTING text]

[STRUCTURE_PRACTICE_QUESTIONS text with formatting guidelines]

Tool context: [empty]
History: [empty]

Thinking context:
=== Question Design Context ===
Topic: American Civil War
Domain: History

Practice Question
[Generated question about Civil War]

[Answer bank with correct answer marked]

User query: Can you give me practice questions on the Civil War?

The current year is 2025...
```

---

### Example 3: Follow-up to Practice Question

**User Input**: "I think the answer is B"

**Active Prompts**:
- CORE_IDENTITY
- GENERAL_FORMATTING
- PRACTICE_QUESTION_FOLLOWUP (Agent 3)

**Thinking Agents**: None

**Final Prompt Structure**:
```
[CORE_IDENTITY text - includes grading instructions]

[GENERAL_FORMATTING text]

[PRACTICE_QUESTION_FOLLOWUP text with grading guidelines]

Tool context: [empty]

History:
assistant: "Which of the following was a major cause of the Civil War?
A. Westward expansion
B. Slavery and states' rights
C. Foreign relations
D. Economic depression"

user: "I think the answer is B"

Thinking context: [empty]

User query: I think the answer is B

The current year is 2025...
```

---

## Prompt Formatting Requirements

### LaTeX Math
```python
# Inline
$x^2 + y^2 = z^2$

# Display (centered)
$$\int_{0}^{1} x^2 \,dx$$

# Literal dollar
\$5.00

# Literal parentheses
\(a + b\)
```

### Practice Question Formats

**Single Choice**:
```
A. Option 1
B. Option 2
C. Option 3
D. Option 4
```

**Multiple Selection**:
```
- [ ] A. Option 1
- [ ] B. Option 2
- [ ] C. Option 3
- [ ] D. Option 4
```

**Written Response**:
```
Please write your response below: [1-2 sentence prompt]
```

### Data Tables
```markdown
| Column 1 | Column 2 | Column 3 |
| :------: | :------: | :------: |
| Data 1   | Data 2   | Data 3   |
```

---

## Prompt Versioning

**Current Version**: v2.0 (Redesign)

**Changes from v1.0**:
- Added thinking prompts (MATH_THINKING, QUESTION_ANSWER_DESIGN, REASONING_THINKING)
- Split routing into 4 specialized agents
- Removed RAG-related prompts
- Added TOOL_USE_ENHANCEMENT
- Restructured agent system prompts

**Compatibility**: All prompts in `prompt_library.py` must be imported into `agents.py`
```

## 6. **MODEL_LOADING.md**

```markdown
# Model Loading Strategy

## Overview

Mimir uses a three-stage model loading strategy:
1. **Build Time**: Download models during Docker build
2. **Startup Time**: Compile and warm up models
3. **Runtime**: Lazy load agents on first GPU call

---

## Stage 1: Build Time (preload_from_hub)

### Configuration (README.md)
```yaml
preload_from_hub:
  - jdesiree/Mimir-Phi-3.5
  - microsoft/Phi-3-mini-4k-instruct
  - yentinglin/Mistral-Small-24B-Instruct-2501-reasoning
  - brittlewis12/Mistral-Small-24B-Instruct-2501-reasoning-GGUF mistral-small-24b-instruct-2501-reasoning-Q4_K_M.gguf
  - thenlper/gte-small
```

### What Happens
```
Docker Build Process
    ↓
HuggingFace Hub downloads models
    ↓
Files saved to: ~/.cache/huggingface/hub/
    ↓
Models ready for Stage 2
```

### Benefits
- No download time during app startup
- Models cached in Docker layer
- Faster cold starts

---

## Stage 2: Startup Time (compile_model.py)

### Purpose
- Load models from HF cache
- Apply quantization
- Run warmup inference
- Create compilation markers

### Process

#### 1. Check Cache Status
```python
def check_model_cache() -> Dict[str, bool]:
    cache_status = {
        "phi3": os.path.exists(f"{CACHE_DIR}/PHI3_READY"),
        "mistral_reasoning": os.path.exists(f"{CACHE_DIR}/MISTRAL_REASONING_READY"),
        "mistral_math_gguf": os.path.exists(f"{CACHE_DIR}/MISTRAL_MATH_GGUF_READY"),
        "all_compiled": os.path.exists(f"{CACHE_DIR}/COMPILED_READY"),
    }
    return cache_status
```

#### 2. Compile Phi-3
```python
def compile_phi3():
    # Try fine-tuned first
    try:
        model = AutoModelForCausalLM.from_pretrained(
            FINE_TUNED_PHI3,
            quantization_config=quantization_config,
            torch_dtype=torch.float16,
            device_map="auto",
        )
    except:
        # Fallback to base
        model = AutoModelForCausalLM.from_pretrained(
            BASE_PHI3,
            quantization_config=quantization_config,
            torch_dtype=torch.float16,
            device_map="auto",
        )
    
    # Warmup
    test_prompt = tokenizer.apply_chat_template(...)
    inputs = tokenizer(test_prompt, return_tensors="pt").to(model.device)
    _ = model.generate(**inputs, max_new_tokens=10)
    
    # Save marker
    with open(f"{CACHE_DIR}/PHI3_READY", "w") as f:
        f.write("Phi-3 model loaded\n")
```

#### 3. Compile Mistral Reasoning
```python
def compile_mistral_reasoning():
    model = AutoModelForCausalLM.from_pretrained(
        MISTRAL_REASONING,
        quantization_config=quantization_config,
        torch_dtype=torch.float16,
        device_map="auto",
    )
    
    # Warmup
    _ = model.generate(**inputs, max_new_tokens=10)
    
    # Save marker
    with open(f"{CACHE_DIR}/MISTRAL_REASONING_READY", "w") as f:
        f.write("Mistral reasoning model loaded\n")
```

#### 4. Compile GGUF Math Model
```python
def compile_mistral_math_gguf():
    # Download GGUF file (already cached from preload_from_hub)
    model_path = hf_hub_download(
        repo_id=MISTRAL_MATH_GGUF,
        filename="mistral-small-24b-instruct-2501-reasoning-Q4_K_M.gguf",
    )
    
    # Test load
    math_model = Llama(
        model_path=model_path,
        n_ctx=4096,
        n_threads=4,
        n_gpu_layers=35,
    )
    
    # Warmup
    _ = math_model("Test prompt", max_tokens=10)
    
    # Save marker with path
    with open(f"{CACHE_DIR}/MISTRAL_MATH_GGUF_READY", "w") as f:
        f.write(f"GGUF model path: {model_path}\n")
```

#### 5. Main Compilation
```python
def compile_all():
    os.makedirs(CACHE_DIR, exist_ok=True)
    
    compile_phi3()
    compile_mistral_reasoning()
    compile_mistral_math_gguf()
    
    # Final marker
    with open(f"{CACHE_DIR}/COMPILED_READY", "w") as f:
        f.write("All models compiled successfully\n")
```

### Integration in app.py
```python
if __name__ == "__main__":
    from compile_model import compile_all
    compile_all()  # Warm up models
    
    # Then create interface
    interface = create_interface()
    interface.launch()
```

---

## Stage 3: Runtime (Lazy Loading)

### Agent Initialization
```python
# agents.py
class ToolDecisionAgent:
    def __init__(self):
        self.model = None
        self.tokenizer = None
        self.model_loaded = False
        logger.info("ToolDecisionAgent initialized (lazy loading)")
```

### First Call Triggers Load
```python
@spaces.GPU(duration=60)
def should_use_visualization(self, query: str) -> bool:
    self._load_model()  # Only loads if not already loaded
    
    # Use model
    outputs = self.model.generate(...)
    return result
```

### Load Model Method
```python
def _load_model(self):
    if self.model_loaded:
        return  # Already loaded, skip
    
    logger.info(f"Loading tool decision model: {MISTRAL_REASONING}")
    
    # Models already in cache from Stage 1+2
    self.tokenizer = AutoTokenizer.from_pretrained(
        MISTRAL_REASONING,
        token=HF_TOKEN
    )
    
    self.model = AutoModelForCausalLM.from_pretrained(
        MISTRAL_REASONING,
        quantization_config=quantization_config,
        torch_dtype=torch.float16,
        device_map="auto",
    )
    
    self.model_loaded = True
```

---

## Model Specifications

### Phi-3 (ResponseAgent)

**Primary**: `jdesiree/Mimir-Phi-3.5` (fine-tuned)
**Fallback**: `microsoft/Phi-3-mini-4k-instruct`

**Configuration**:
```python
quantization_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_compute_dtype=torch.float16,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_use_double_quant=True,
)

model = AutoModelForCausalLM.from_pretrained(
    model_path,
    quantization_config=quantization_config,
    torch_dtype=torch.float16,
    trust_remote_code=True,
    low_cpu_mem_usage=True,
    attn_implementation="eager",
    device_map="auto",
)
```

**Accelerate Integration**:
```python
accelerator = Accelerator(
    mixed_precision="fp16",
    gradient_accumulation_steps=1,
)
model = accelerator.prepare(model)
```

**Generation Parameters**:
```python
outputs = model.generate(
    input_ids=inputs['input_ids'],
    max_new_tokens=350,
    do_sample=True,
    temperature=0.7,
    repetition_penalty=1.15,
    pad_token_id=tokenizer.eos_token_id,
)
```

---

### Mistral-Small-24B (Routing & Thinking Agents)

**Model**: `yentinglin/Mistral-Small-24B-Instruct-2501-reasoning`

**Shared Usage**:
- ToolDecisionAgent
- All 4 PromptRoutingAgents
- ThinkingAgents (QA Design, Reasoning)

**Configuration**:
```python
quantization_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_compute_dtype=torch.float16,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_use_double_quant=True,
)

model = AutoModelForCausalLM.from_pretrained(
    MISTRAL_REASONING,
    quantization_config=quantization_config,
    torch_dtype=torch.float16,
    trust_remote_code=True,
    low_cpu_mem_usage=True,
    device_map="auto",
)
```

**Generation Parameters** (vary by agent):
```python
# Tool Decision (short)
outputs = model.generate(**inputs, max_new_tokens=10, temperature=0.1)

# Routing Agents (medium)
outputs = model.generate(**inputs, max_new_tokens=30-50, temperature=0.1)

# Thinking Agents (long)
outputs = model.generate(**inputs, max_new_tokens=512, temperature=0.7)
```

---

### GGUF Mistral (Math Thinking)

**Model**: `brittlewis12/Mistral-Small-24B-Instruct-2501-reasoning-GGUF`
**File**: `mistral-small-24b-instruct-2501-reasoning-Q4_K_M.gguf`

**Configuration**:
```python
math_model = Llama(
    model_path=model_path,
    n_ctx=4096,
    n_threads=4,
    n_gpu_layers=35,
)
```

**Generation**:
```python
response = math_model(
    full_prompt,
    max_tokens=512,
    temperature=0.7,
    stop=["</thinking>", "\n\n---", "<|end|>"],
)
```

**Cache-Aware Loading**:
```python
def get_cached_gguf_path() -> Optional[str]:
    marker_file = f"{CACHE_DIR}/MISTRAL_MATH_GGUF_READY"
    if os.path.exists(marker_file):
        with open(marker_file, 'r') as f:
            content = f.read()
            if "GGUF model path:" in content:
                path = content.split("GGUF model path:")[-1].strip()
                if os.path.exists(path):
                    return path
    return None

# Usage in agent
cached_path = get_cached_gguf_path()
if cached_path:
    model_path = cached_path  # Fast path
else:
    model_path = hf_hub_download(...)  # Download
```

---

## ZeroGPU Integration

### Decorator Usage
```python
@spaces.GPU(duration=60)  # Allocate GPU for 60 seconds
def agent_method(self, input):
    # GPU operations here
    return result
```

### GPU Duration by Agent

| Agent | Duration | Reason |
|-------|----------|--------|
| ToolDecisionAgent | 60s | Short inference (10 tokens) |
| PromptRoutingAgents | 50s each | Medium inference (30-50 tokens) |
| ThinkingAgents.math_thinking | 60s | Long inference (512 tokens) |
| ThinkingAgents.qa_design | 60s | Long inference (512 tokens) |
| ThinkingAgents.reasoning | 60s | Long inference (512 tokens) |
| ResponseAgent | 120s | Longest inference (350 tokens) + post-processing |

### GPU Allocation Strategy
```
User turn starts
    ↓
@spaces.GPU triggers for each agent in sequence
    ↓
GPU allocated → inference → GPU released
    ↓
Next agent
    ↓
Repeat until ResponseAgent completes
```

---

## Memory Optimization

### 4-bit Quantization
- Reduces memory footprint by ~75%
- Phi-3: ~3.8B params → ~1GB VRAM (4-bit)
- Mistral: ~24B params → ~6GB VRAM (4-bit)

### BitsAndBytes Configuration
```python
BitsAndBytesConfig(
    load_in_4bit=True,                    # Use 4-bit precision
    bnb_4bit_compute_dtype=torch.float16, # Compute in FP16
    bnb_4bit_quant_type="nf4",            # Normal Float 4
    bnb_4bit_use_double_quant=True,       # Double quantization
)
```

### Device Mapping
```python
device_map="auto"  # Automatically distribute model across available devices
```

---

## Cache Directory Structure

```
/data/compiled_models/
├── PHI3_READY                    # Marker: Phi-3 compiled
├── MISTRAL_REASONING_READY       # Marker: Mistral compiled
├── MISTRAL_MATH_GGUF_READY       # Marker: GGUF path
├── RAG_EMBEDDINGS_READY          # Marker: Embeddings compiled
└── COMPILED_READY                # Marker: All compiled
```

**Marker File Example** (`MISTRAL_MATH_GGUF_READY`):
```
GGUF model path: /home/user/.cache/huggingface/hub/models--brittlewis12--Mistral-Small-24B-Instruct-2501-reasoning-GGUF/snapshots/abc123/mistral-small-24b-instruct-2501-reasoning-Q4_K_M.gguf
```

---

## Fallback Strategies

### Phi-3 Fine-tuned → Base
```python
try:
    model = load_fine_tuned_model()
    self.model_type = "fine-tuned"
except Exception as e:
    logger.warning(f"Fine-tuned failed: {e}, using base")
    model = load_base_model()
    self.model_type = "base-fallback"
```

### GGUF Not Available
```python
if not LLAMA_CPP_AVAILABLE:
    logger.warning("llama-cpp-python not available - GGUF disabled")
    # Math thinking returns empty string
    return ""
```

---

## Performance Optimization

### Lazy Loading Benefits
- Only load models when needed
- Reduce startup time
- Save memory for unused agents

### Shared Model Strategy
- RoutingAgents share ONE Mistral instance
- Reduces memory by 5x (vs loading 4 separate instances)

### Warmup Inference
- Compile models at startup (compile_model.py)
- First real inference is faster
- Creates CUDA kernels ahead of time

---

## Troubleshooting

### Model Not Loading
1. Check HF cache: `~/.cache/huggingface/hub/`
2. Verify token: `echo $HF_TOKEN`
3. Check markers: `ls /data/compiled_models/`

### Out of Memory
1. Reduce GPU layers for GGUF: `n_gpu_layers=20`
2. Reduce max_new_tokens
3. Clear CUDA cache: `torch.cuda.empty_cache()`

### Slow First Inference
- Normal! First call compiles CUDA kernels
- Subsequent calls are faster
- Run `compile_model.py` to pre-warm
```

## 7. **DEPLOYMENT.md**

```markdown
# Deployment Guide - HuggingFace Spaces

## Overview

Mimir is deployed on HuggingFace Spaces with ZeroGPU support.

**Space URL**: `https://huggingface.co/spaces/jdesiree/Mimir`

---

## Space Configuration

### README.md Metadata

```yaml
---
title: Mimir
emoji: 📚
colorFrom: indigo
colorTo: blue
sdk: gradio
sdk_version: 5.49.1
app_file: app.py
pinned: true
python_version: '3.10'
short_description: Advanced prompt engineering for educational AI systems.
thumbnail: >-
  https://cdn-uploads.huggingface.co/production/uploads/68700e7552b74a1dcbb2a87e/Z7P8DJ57rc5P1ozA5gwp3.png
hardware: zero-gpu-dynamic
startup_duration_timeout: 1h
hf_oauth: true
hf_oauth_expiration_minutes: 180
preload_from_hub:
  - jdesiree/Mimir-Phi-3.5
  - microsoft/Phi-3-mini-4k-instruct
  - yentinglin/Mistral-Small-24B-Instruct-2501-reasoning
  - brittlewis12/Mistral-Small-24B-Instruct-2501-reasoning-GGUF mistral-small-24b-instruct-2501-reasoning-Q4_K_M.gguf
  - thenlper/gte-small
---
```

### Key Settings Explained

#### Hardware
```yaml
hardware: zero-gpu-dynamic
```
- **Dynamic GPU allocation**: GPU assigned only when `@spaces.GPU` decorator triggered
- **No persistent GPU**: Saves costs, but adds latency
- **Alternative**: `zero-gpu-persistent` (faster but more expensive)

#### Startup Timeout
```yaml
startup_duration_timeout: 1h
```
- Allows up to 1 hour for startup (model compilation)
- Default is 30 minutes
- Increase if `compile_model.py` takes longer

#### OAuth
```yaml
hf_oauth: true
hf_oauth_expiration_minutes: 180
```
- Requires user login via HuggingFace
- Session expires after 3 hours
- Access user info: `request.username`

#### Model Preloading
```yaml
preload_from_hub:
  - jdesiree/Mimir-Phi-3.5              # Full repo
  - microsoft/Phi-3-mini-4k-instruct    # Full repo
  - yentinglin/...                      # Full repo
  - brittlewis12/...GGUF file.gguf      # Specific file
  - thenlper/gte-small                  # Full repo
```
- Downloads during Docker build
- Cached in `~/.cache/huggingface/hub/`
- Specific file syntax: `repo_id filename.ext`

---

## File Structure

```
mimir/
├── README.md                      # Space configuration
├── requirements.txt               # Python dependencies
├── app.py                         # Main application
├── agents.py                      # Agent architecture
├── state_manager.py               # State management
├── prompt_library.py              # Prompt templates
├── graph_tool.py                  # Visualization tool
├── compile_model.py               # Model compilation script
├── gradio_chatbot.py              # Chatbot interface
├── gradio_analytics.py            # Analytics interface
├── loading_animation.gif          # Loading spinner
├── favicon.ico                    # Browser icon
├── .env                           # Environment variables (HF_TOKEN)
├── wheels/                        # (Optional) Prebuilt wheels
│   └── llama_cpp_python-*.whl
└── mimir_analytics.db             # SQLite database (created at runtime)
```

---

## requirements.txt

```txt
# ZeroGPU COMPATIBILITY
spaces

# CORE ML/AI PACKAGES
transformers>=4.41.0
huggingface_hub>=0.20.0
safetensors
accelerate>=0.31.0
bitsandbytes
sentencepiece
peft>=0.10.0

# GGUF model support for Math Thinking Agent
llama-cpp-python>=0.3.16

# LANGCHAIN ECOSYSTEM
langgraph>=0.2.0
langchain-core>=0.3.0
langchain-community>=0.3.0
langchain-huggingface>=0.1.0

# UI FRAMEWORK
gradio>=5.46.1

# DATA & STATE MANAGEMENT
datasets>=2.14.0
python-dotenv>=1.0.0

# VISUALIZATION & TOOLS
matplotlib>=3.7.0
plotly>=5.15.0
pandas>=2.0.0
numpy>=1.24.0

# METRICS & EVALUATION
lighteval
trackio

# UTILITIES
tqdm>=4.65.0
```

**Important**:
- `spaces` must be first (ZeroGPU compatibility)
- No torch version specified (provided by ZeroGPU environment)
- `llama-cpp-python` installed from PyPI (not prebuilt wheel)

---

## Environment Variables

### .env File
```bash
HF_TOKEN=hf_xxxxxxxxxxxxxxxxxxxxx
HUGGINGFACEHUB_API_TOKEN=hf_xxxxxxxxxxxxxxxxxxxxx  # Fallback
DEBUG_STATE=false
```

### Space Secrets
Set in HuggingFace Space settings:
1. Go to Space → Settings → Variables and secrets
2. Add `HF_TOKEN` as a secret
3. Value: Your HuggingFace access token

**Token Permissions Required**:
- Read access to model repos
- Write access to dataset repo (`jdesiree/mimir_analytics`)

---

## Build Process

### 1. Docker Build Stage

```dockerfile
# Automatic by HF Spaces
FROM python:3.10

# Install dependencies
RUN pip install -r requirements.txt

# Preload models (from preload_from_hub)
RUN huggingface-cli download jdesiree/Mimir-Phi-3.5
RUN huggingface-cli download microsoft/Phi-3-mini-4k-instruct
RUN huggingface-cli download yentinglin/Mistral-Small-24B-Instruct-2501-reasoning
RUN huggingface-cli download brittlewis12/.../GGUF mistral...gguf
RUN huggingface-cli download thenlper/gte-small

# Copy application code
COPY . /app
WORKDIR /app

# Start application
CMD ["python", "app.py"]
```

### 2. Startup Stage (app.py)

```python
if __name__ == "__main__":
    from compile_model import compile_all
    
    # Compile and warm up models
    compile_all()
    
    # Create Gradio interface
    interface = create_interface()
    
    # Launch
    interface.launch(
        server_name="0.0.0.0",
        server_port=7860,  # HF Spaces default
        share=False,
        debug=False,
    )
```

---

## Deployment Workflow

### 1. Local Development

```bash
# Clone repo
git clone https://huggingface.co/spaces/jdesiree/Mimir
cd Mimir

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Set environment variable
export HF_TOKEN=your_token_here

# Run locally
python app.py
```

### 2. Push to HuggingFace

```bash
# Initialize git (if not already)
git init
git remote add origin https://huggingface.co/spaces/jdesiree/Mimir

# Stage changes
git add .

# Commit
git commit -m "Update: [describe changes]"

# Push
git push origin main
```

**Automatic Rebuild**: Space rebuilds automatically on push

### 3. Monitor Build

1. Go to Space page
2. Click "Logs" tab
3. Watch build progress
4. Common stages:
   - Installing dependencies (5-10 min)
   - Preloading models (10-20 min)
   - Compiling models (5-10 min)
   - Starting Gradio (< 1 min)

### 4. Verify Deployment

1. Space status shows "Running" (green)
2. Visit Space URL
3. Test basic interaction:
   - Send "Hello"
   - Verify response
   - Check Analytics page

---

## Troubleshooting

### Build Timeout

**Symptom**: Space fails to build within 1 hour

**Solutions**:
1. Increase timeout in README.md:
   ```yaml
   startup_duration_timeout: 2h
   ```
2. Skip `compile_model.py` during build (lazy load only)
3. Reduce model preloading

---

### Out of Memory

**Symptom**: `CUDA out of memory` errors

**Solutions**:
1. Reduce `n_gpu_layers` in GGUF loading:
   ```python
   math_model = Llama(..., n_gpu_layers=20)  # Was 35
   ```
2. Reduce `max_new_tokens` in generation:
   ```python
   outputs = model.generate(..., max_new_tokens=250)  # Was 350
   ```
3. Clear cache between generations:
   ```python
   torch.cuda.empty_cache()
   ```

---

### Model Not Found

**Symptom**: `Repository not found` or `File not found`

**Solutions**:
1. Verify model repos are public or token has access
2. Check `preload_from_hub` syntax:
   ```yaml
   # Correct
   - brittlewis12/Model-GGUF model.gguf
   
   # Incorrect
   - brittlewis12/Model-GGUF/model.gguf
   ```
3. Verify HF_TOKEN secret is set

---

### Space Crashes on Startup

**Symptom**: Space shows "Runtime Error"

**Solutions**:
1. Check logs for error messages
2. Test locally first
3. Common issues:
   - Missing `.env` file → Use Space secrets
   - Import errors → Check `requirements.txt`
   - Syntax errors → Test with `python -m py_compile app.py`

---

### Slow Response Time

**Symptom**: 30+ second response times

**Expected**: This is normal for ZeroGPU due to:
- Dynamic GPU allocation overhead (~5-10s per agent)
- Multiple agent calls per turn
- Model quantization/dequantization

**Optimization**:
1. Use `zero-gpu-persistent` for faster GPU access:
   ```yaml
   hardware: zero-gpu-persistent
   ```
2. Reduce number of agent calls:
   - Skip unnecessary routing agents
   - Combine thinking agents
3. Adjust `@spaces.GPU` durations:
   ```python
   @spaces.GPU(duration=30)  # Reduce from 60
   ```

---

## Monitoring & Maintenance

### Space Logs

**Access**: Space page → Logs tab

**What to Monitor**:
- Build errors
- Runtime exceptions
- GPU allocation times
- User interaction logs

### Analytics Dashboard

**Access**: In-app → Analytics page

**Metrics**:
- Total conversations
- Average response time
- Success rate
- Recent interactions

### Database Backup

**Automatic**: HF Dataset backup every hour
- Repo: `jdesiree/mimir_analytics`
- Format: Parquet
- Access: `datasets.load_dataset("jdesiree/mimir_analytics")`

**Manual Backup**:
```python
from state_manager import GlobalStateManager
global_state_manager = GlobalStateManager()
global_state_manager._backup_to_hf_dataset()
```

## Support & Resources

### HuggingFace Docs
- [Spaces Overview](https://huggingface.co/docs/hub/spaces)
- [ZeroGPU Guide](https://huggingface.co/docs/hub/spaces-zerogpu)
- [Gradio on Spaces](https://huggingface.co/docs/hub/spaces-sdks-gradio)

### Mimir Resources
- [Space URL](https://huggingface.co/spaces/jdesiree/Mimir)
- [Model Repo](https://huggingface.co/jdesiree/Mimir-Phi-3.5)
- [Analytics Dataset](https://huggingface.co/datasets/jdesiree/mimir_analytics)
- [Case Study](https://github.com/Jdesiree112/Technical_Portfolio/tree/main/CaseStudy_Mimir)
- [Corresponding API Reference](https://github.com/Jdesiree112/Technical_Portfolio/tree/main/CaseStudy_Mimir/API_Docs)

### Contact
- Space discussions: Comment on the [Space page](https://huggingface.co/spaces/jdesiree/Mimir/discussions/new)
- Issues: [GitHub Issues](https://github.com/Jdesiree112/Technical_Portfolio/tree/main/CaseStudy_Mimir/Issue_ProblemSolvingCaseStudy)
- Email: jdesiree112@gmail.com
```
