# Mimir Educational AI Assistant - API Integration Guide

**Version:** 2.0  
**Created:** October 2025  
**Platform:** Hugging Face Spaces (Gradio)  
**Space:** `jdesiree/Mimir`  
**Architecture:** Multi-Agent Educational System

## Overview

Mimir is an advanced educational AI assistant hosted on Hugging Face Spaces, built with a multi-agent architecture and Gradio interface. The system employs specialized agents for tool decisions, prompt routing, complex reasoning, and response generation, all orchestrated through a state management system with SQLite and HuggingFace Dataset persistence.

**Key Features:**
- Multi-agent decision pipeline (Tool, Routing, Thinking, Response)
- Dynamic prompt assembly based on educational context
- ZeroGPU integration for efficient resource usage
- Per-turn state management with conversation persistence
- Dual-page interface (Chatbot + Analytics)

## Quick Start

### Installation

```bash
pip install gradio_client
```

### Basic Connection

```python
from gradio_client import Client

# Connect to Mimir Space
client = Client("jdesiree/Mimir")
```

### Authentication (if OAuth enabled)

```python
# If Space requires HuggingFace OAuth
from huggingface_hub import login

login(token="your_hf_token")
client = Client("jdesiree/Mimir")
```

## API Endpoints

### Message Handling

#### `/add_user_message`
Adds user message to conversation state and updates chat display.

**Parameters:**
- `message` (str, required): The user input message
- `chat_history` (list, optional, default: []): Current chat history for display
- `conversation_state` (list, optional, default: []): Internal conversation state

**Returns:**
- Tuple of 3 elements:
  - [0] str: Empty string (cleared input box)
  - [1] list: Updated chat_history (for display)
  - [2] list: Updated conversation_state (for agent context)

**Example:**
```python
result = client.predict(
    message="Explain the Pythagorean theorem",
    chat_history=[],
    conversation_state=[],
    api_name="/add_user_message"
)
cleared_input, chat_history, conversation_state = result
```

**Notes:**
- Updates both display state (chat_history) and agent context (conversation_state)
- Persists to GlobalStateManager (SQLite + HF Dataset)
- Input box is automatically cleared after submission

---

#### `/add_user_message_1`
Duplicate endpoint - identical functionality to `/add_user_message`.

---

### Response Generation

#### `/generate_response`
Orchestrates multi-agent pipeline to generate educational response with streaming output.

**Parameters:**
- `chat_history` (list, optional, default: []): Current chat display state
- `conversation_state` (list, optional, default: []): Internal conversation context

**Returns:**
- Tuple of 2 elements:
  - [0] list: Updated chat_history with AI response
  - [1] list: Updated conversation_state with AI response

**Processing Pipeline:**
1. **Step 1**: Reset prompt state for new turn
2. **Step 2**: Process user input and retrieve history
3. **Step 3**: Tool Decision Agent determines visualization needs
4. **Step 4**: Regex-based logical expressions check keywords
5. **Step 5**: Sequential Routing Agents (4 agents) select prompts
6. **Step 6**: Thinking Agents (if activated) generate reasoning context
7. **Step 7**: Assemble response prompts dynamically
8. **Step 8**: Construct complete prompt with all context
9. **Step 9**: Response Agent (Phi-3) generates response
10. **Step 10**: Post-process response (clean, truncate, enhance)
11. **Step 11**: Track metrics (educational quality, timing)

**Example:**
```python
# After adding user message
response = client.predict(
    chat_history=chat_history,
    conversation_state=conversation_state,
    api_name="/generate_response"
)
updated_chat, updated_state = response
```

**Performance Notes:**
- Typical response time: 25-40 seconds
- Uses ZeroGPU with dynamic allocation
- Streams response word-by-word for better UX
- Multiple GPU allocations per turn (1 per agent call)

---

#### `/generate_response_1`
Duplicate endpoint - identical functionality to `/generate_response`.

---

### UI Enhancement

#### `/add_loading_animation`
Adds animated loading indicator to chat display.

**Parameters:**
- `chat_history` (list, optional, default: []): Current chat display state
- `conversation_state` (list, optional, default: []): Internal conversation state

**Returns:**
- Tuple of 2 elements:
  - [0] list: Chat history with loading animation appended
  - [1] list: Unchanged conversation_state

**Example:**
```python
loading_display = client.predict(
    chat_history=chat_history,
    conversation_state=conversation_state,
    api_name="/add_loading_animation"
)
```

**Notes:**
- Loading animation is display-only (GIF)
- Does not modify conversation_state
- Automatically removed when response generation begins
- Falls back to empty div if `loading_animation.gif` not found

---

#### `/add_loading_animation_1`
Duplicate endpoint - identical functionality to `/add_loading_animation`.

---

### Session Management

#### `/reset_conversation`
Resets conversation to initial state, clearing all history.

**Parameters:**
- None

**Returns:**
- Tuple of 2 elements:
  - [0] list: Empty chat_history
  - [1] list: Empty conversation_state

**Example:**
```python
reset_state = client.predict(api_name="/reset_conversation")
empty_chat, empty_state = reset_state
```

**Notes:**
- Calls `GlobalStateManager.reset_conversation_state()`
- Clears both display and internal state
- Maintains session_id for analytics tracking
- Does not affect analytics cache

---

### Analytics (Dashboard Page)

#### `/refresh_analytics`
Refreshes analytics dashboard with latest metrics.

**Parameters:**
- None

**Returns:**
- Tuple of 3 elements:
  - [0] dict: Project statistics
  - [1] list: Recent interactions data
  - [2] str: Dashboard HTML

**Example:**
```python
analytics = client.predict(api_name="/refresh_analytics")
project_stats, recent_interactions, dashboard_html = analytics
```

**Available Metrics:**
- Total conversations
- Average response time
- Success rate
- Educational quality scores
- Classifier accuracy
- Active sessions

---

## Data Structures

### Chat History Structure

Display state for Gradio Chatbot component:

```python
chat_history: list[dict(
    role: Literal['user', 'assistant'],
    content: str,  # Message content (may include HTML/Markdown)
    metadata: dict | None  # Optional metadata
)]
```

**Example:**
```python
chat_history = [
    {"role": "user", "content": "What is calculus?"},
    {"role": "assistant", "content": "Calculus is a branch of mathematics..."}
]
```

---

### Conversation State Structure

Internal state for agent context (simplified view):

```python
conversation_state: list[dict(
    role: Literal['user', 'assistant'],
    content: str  # Pure text content (no formatting)
)]
```

**Differences from chat_history:**
- No HTML/Markdown formatting
- No loading animations
- Used by agents for context
- Stored in GlobalStateManager

---

### Project Statistics Structure

```python
project_stats: dict(
    total_conversations: int | None,
    avg_session_length: float | None,  # seconds
    success_rate: float | None,  # percentage
    ml_educational_quality: float | None,  # 0.0-1.0
    ml_classifier_accuracy: float | None,  # percentage
    active_sessions: int | None,
    model_type: str,  # "Phi-3-mini (Fine-tuned)"
    last_updated: str  # ISO datetime
)
```

---

## Integration Patterns

### 1. Simple Conversation Flow

```python
from gradio_client import Client

client = Client("jdesiree/Mimir")

# Reset to start fresh
client.predict(api_name="/reset_conversation")

# Initialize state
chat_history = []
conversation_state = []

# Add user message
result = client.predict(
    message="Explain photosynthesis",
    chat_history=chat_history,
    conversation_state=conversation_state,
    api_name="/add_user_message"
)
_, chat_history, conversation_state = result

# Generate AI response
response = client.predict(
    chat_history=chat_history,
    conversation_state=conversation_state,
    api_name="/generate_response"
)
chat_history, conversation_state = response

# Extract final response
final_message = chat_history[-1]['content']
print(final_message)
```

---

### 2. Conversation with Loading States

```python
# Add user message
result = client.predict(
    message="What is the derivative of x^2?",
    chat_history=chat_history,
    conversation_state=conversation_state,
    api_name="/add_user_message"
)
_, chat_history, conversation_state = result

# Show loading animation
loading_result = client.predict(
    chat_history=chat_history,
    conversation_state=conversation_state,
    api_name="/add_loading_animation"
)
display_with_loading = loading_result[0]

# Generate response (loading animation auto-removed)
response = client.predict(
    chat_history=chat_history,  # Use original, not loading state
    conversation_state=conversation_state,
    api_name="/generate_response"
)
chat_history, conversation_state = response
```

---

### 3. Multi-turn Conversation with Context

```python
chat_history = []
conversation_state = []

# Turn 1: Initial question
result = client.predict(
    message="I need help with quadratic equations",
    chat_history=chat_history,
    conversation_state=conversation_state,
    api_name="/add_user_message"
)
_, chat_history, conversation_state = result

response = client.predict(
    chat_history=chat_history,
    conversation_state=conversation_state,
    api_name="/generate_response"
)
chat_history, conversation_state = response

# Turn 2: Follow-up (with context)
result = client.predict(
    message="Can you give me a practice problem?",
    chat_history=chat_history,
    conversation_state=conversation_state,
    api_name="/add_user_message"
)
_, chat_history, conversation_state = result

response = client.predict(
    chat_history=chat_history,
    conversation_state=conversation_state,
    api_name="/generate_response"
)
chat_history, conversation_state = response

# Turn 3: Answer practice question
result = client.predict(
    message="I think the answer is x = 3 and x = -2",
    chat_history=chat_history,
    conversation_state=conversation_state,
    api_name="/add_user_message"
)
_, chat_history, conversation_state = result

response = client.predict(
    chat_history=chat_history,
    conversation_state=conversation_state,
    api_name="/generate_response"
)
chat_history, conversation_state = response
# Response will include grading and feedback
```

---

### 4. Analytics Monitoring

```python
# Get current analytics
analytics = client.predict(api_name="/refresh_analytics")
stats, interactions, html = analytics

print(f"Total Conversations: {stats['total_conversations']}")
print(f"Average Response Time: {stats['avg_session_length']}s")
print(f"Success Rate: {stats['success_rate']}%")
print(f"Educational Quality: {stats['ml_educational_quality']}")

# Display recent interactions
for interaction in interactions[:5]:
    timestamp, response_time, mode, tools, quality, adapter = interaction
    print(f"{timestamp}: {response_time}s, Quality: {quality}")
```

---

### 5. Session Persistence

```python
import json

# Save session state
session_data = {
    'chat_history': chat_history,
    'conversation_state': conversation_state,
    'timestamp': datetime.now().isoformat()
}

with open('session.json', 'w') as f:
    json.dump(session_data, f)

# Restore session
with open('session.json', 'r') as f:
    session_data = json.load(f)

chat_history = session_data['chat_history']
conversation_state = session_data['conversation_state']

# Continue conversation
result = client.predict(
    message="Let's continue where we left off",
    chat_history=chat_history,
    conversation_state=conversation_state,
    api_name="/add_user_message"
)
```

---

## Advanced Usage

### Error Handling

```python
def safe_conversation(client, message, chat_history=None, conversation_state=None):
    """Wrapper with comprehensive error handling"""
    if chat_history is None:
        chat_history = []
    if conversation_state is None:
        conversation_state = []
    
    try:
        # Add user message
        result = client.predict(
            message=message,
            chat_history=chat_history,
            conversation_state=conversation_state,
            api_name="/add_user_message"
        )
        _, chat_history, conversation_state = result
        
        # Generate response
        response = client.predict(
            chat_history=chat_history,
            conversation_state=conversation_state,
            api_name="/generate_response"
        )
        chat_history, conversation_state = response
        
        return chat_history, conversation_state, None
        
    except Exception as e:
        error_msg = f"API call failed: {e}"
        print(error_msg)
        return chat_history, conversation_state, error_msg
```

---

### Timeout Handling

```python
import time
from gradio_client import Client

def conversation_with_timeout(client, message, chat_history, conversation_state, timeout=60):
    """Generate response with timeout"""
    import concurrent.futures
    
    with concurrent.futures.ThreadPoolExecutor() as executor:
        # Submit API call
        future = executor.submit(
            client.predict,
            message=message,
            chat_history=chat_history,
            conversation_state=conversation_state,
            api_name="/add_user_message"
        )
        
        try:
            result = future.result(timeout=timeout)
            _, chat_history, conversation_state = result
            
            # Generate response
            response = client.predict(
                chat_history=chat_history,
                conversation_state=conversation_state,
                api_name="/generate_response"
            )
            return response
            
        except concurrent.futures.TimeoutError:
            print(f"Request timed out after {timeout}s")
            return chat_history, conversation_state
```

---

### Batch Processing

```python
def process_questions_batch(client, questions):
    """Process multiple questions in sequence"""
    results = []
    
    for question in questions:
        # Start fresh for each question
        client.predict(api_name="/reset_conversation")
        
        chat_history = []
        conversation_state = []
        
        # Process question
        result = client.predict(
            message=question,
            chat_history=chat_history,
            conversation_state=conversation_state,
            api_name="/add_user_message"
        )
        _, chat_history, conversation_state = result
        
        response = client.predict(
            chat_history=chat_history,
            conversation_state=conversation_state,
            api_name="/generate_response"
        )
        chat_history, conversation_state = response
        
        # Store result
        results.append({
            'question': question,
            'answer': chat_history[-1]['content'],
            'full_history': chat_history
        })
        
        time.sleep(2)  # Rate limiting
    
    return results
```

---

## Performance Considerations

### Response Time Breakdown

Typical turn processing time: **25-40 seconds**

| Step | Component | Duration | GPU |
|------|-----------|----------|-----|
| 1-2 | State reset & input processing | < 0.2s | No |
| 3 | Tool Decision Agent | 3-5s | Yes (60s alloc) |
| 4 | Regex checks | < 0.01s | No |
| 5 | Routing Agents (4x) | 12-20s | Yes (50s × 4) |
| 6 | Thinking Agents (if active) | 4-8s | Yes (60s each) |
| 7-8 | Prompt assembly | < 0.2s | No |
| 9 | Response Agent | 5-10s | Yes (120s alloc) |
| 10 | Post-processing & streaming | 0.5-1s | No |
| 11 | Metrics tracking | < 0.5s | No |

### Optimization Tips

1. **Minimize unnecessary calls**: Use conversation_state to maintain context
2. **Batch operations**: Reset conversation between unrelated questions
3. **Handle async**: Use threading/async for non-blocking calls
4. **Cache analytics**: Refresh dashboard data only when needed
5. **Rate limiting**: Add delays between batch requests (2-3s recommended)

---

### ZeroGPU Behavior

- **Dynamic allocation**: GPU assigned only during `@spaces.GPU` decorated functions
- **Sequential processing**: Agents run one at a time (not parallel)
- **Cold start**: First call after idle may be slower (~10-15s extra)
- **Warm state**: Subsequent calls faster due to cached models

---

## Limitations & Notes

### API Limitations

- **Duplicate endpoints**: `/add_user_message_1`, `/generate_response_1`, `/add_loading_animation_1` are exact duplicates
- **No async streaming**: Responses return complete, not streamed to API client
- **Session management**: Session IDs managed internally, not exposed via API
- **Rate limiting**: Shared ZeroGPU resources may cause queuing during high traffic
- **OAuth**: If enabled, requires HuggingFace authentication

### Best Practices

1. **Always maintain both states**: Track both `chat_history` and `conversation_state`
2. **Reset when appropriate**: Clear state between unrelated conversations
3. **Handle timeouts**: Agent cascade can take 40+ seconds
4. **Check for errors**: Wrap calls in try-except blocks
5. **Respect display vs. context**: Use chat_history for display, conversation_state for context

### Known Issues

- Loading animations don't stream to API clients (display only)
- Very long conversations (50+ turns) may slow down due to context length
- SQLite database not directly accessible via API
- Analytics export (JSON/CSV) not available via API endpoints

---

## Changes from v1.2 to v2.0

### Major Architectural Changes

#### 1. Multi-Agent System
**v1.2**: Single-model architecture with ML classifier
```python
# Old approach
model.generate(prompt)
```

**v2.0**: Four-stage agent pipeline
```python
# New approach
tool_decision → routing_agents (4x) → thinking_agents (3x) → response_agent
```

**Impact on API**:
- Longer response times (25-40s vs 10-15s)
- More sophisticated prompt selection
- Better educational quality

---

#### 2. State Management

**v1.2**: Simple conversation list
```python
conversation_history = [...]
```

**v2.0**: Dual state system with persistence
```python
chat_history = [...]          # Display state
conversation_state = [...]    # Agent context
# Backed by SQLite + HF Dataset
```

**Impact on API**:
- Must track both `chat_history` and `conversation_state`
- State automatically persisted to database
- Analytics available via `/refresh_analytics`

---

#### 3. Prompt System

**v1.2**: Static prompt templates
- Single system prompt per mode

**v2.0**: Dynamic prompt assembly
- 11 prompt segments activated per-turn
- Logical expressions (regex-based)
- Agent-driven selection

**Impact on API**:
- More contextually appropriate responses
- Better handling of math, practice questions, follow-ups
- Transparent to API users (internal optimization)

---

#### 4. Model Loading

**v1.2**: Single model loaded at startup

**v2.0**: Three-stage loading
1. Build time: `preload_from_hub` downloads models
2. Startup: `compile_model.py` compiles and warms up
3. Runtime: Lazy loading on first agent call

**Impact on API**:
- Slower first request per agent (~5-10s extra)
- Subsequent requests faster
- Cold starts after idle period

---

#### 5. Thinking Agents (NEW)

**v2.0 Addition**: Preprocessing agents for complex reasoning
- `MATH_THINKING`: Tree-of-Thought for math problems
- `QUESTION_ANSWER_DESIGN`: Chain-of-Thought for practice questions
- `REASONING_THINKING`: General CoT preprocessing

**Impact on API**:
- Better quality for complex queries
- Additional processing time (4-8s when activated)
- More structured, educational responses

---

### API Endpoint Changes

#### Modified Endpoints

**`/add_user_message`**
- **v1.2**: `(message, chat_history) → (str, list)`
- **v2.0**: `(message, chat_history, conversation_state) → (str, list, list)`
- **Change**: Added `conversation_state` parameter and return value

**`/generate_response`**
- **v1.2**: `(chat_history) → list`
- **v2.0**: `(chat_history, conversation_state) → (list, list)`
- **Change**: Added `conversation_state` parameter and return value

**`/add_loading_animation`**
- **v1.2**: `(chat_history) → list`
- **v2.0**: `(chat_history, conversation_state) → (list, list)`
- **Change**: Added `conversation_state` (passthrough only)

**`/reset_conversation`**
- **v1.2**: `() → list`
- **v2.0**: `() → (list, list)`
- **Change**: Now returns both chat_history and conversation_state

#### New Endpoints

**`/refresh_analytics`** (NEW)
- Returns project statistics, recent interactions, dashboard HTML
- Access to metrics: response times, quality scores, success rates

---

### Performance Impact

| Metric | v1.2 | v2.0 | Change |
|--------|------|------|--------|
| Avg Response Time | 10-15s | 25-40s | +15-25s |
| First Call (Cold) | 12-18s | 35-50s | +23-32s |
| Educational Quality | 0.65 | 0.82 | +26% |
| Context Awareness | Limited | High | Significant |
| Math Problem Quality | Fair | Excellent | Major improvement |

---

### Migration Guide

#### Updating v1.2 Code to v2.0

**Before (v1.2)**:
```python
# Add user message
_, chat_history = client.predict(
    message="Hello",
    chat_history=[],
    api_name="/add_user_message"
)

# Generate response
chat_history = client.predict(
    chat_history=chat_history,
    api_name="/generate_response"
)
```

**After (v2.0)**:
```python
# Add user message
_, chat_history, conversation_state = client.predict(
    message="Hello",
    chat_history=[],
    conversation_state=[],  # NEW
    api_name="/add_user_message"
)

# Generate response
chat_history, conversation_state = client.predict(  # NEW return value
    chat_history=chat_history,
    conversation_state=conversation_state,  # NEW
    api_name="/generate_response"
)
```

#### State Management Migration

```python
# v1.2 state tracking
chat_history = []

# v2.0 state tracking
chat_history = []          # For display
conversation_state = []    # For agent context

# Both must be maintained throughout conversation
```

---

### Backward Compatibility

**Breaking Changes**:
- All endpoints now require `conversation_state` parameter
- Return values changed from single list to tuple of lists
- Old v1.2 code will **fail** with TypeError

**No Changes**:
- Endpoint names remain the same
- Chat history structure unchanged
- Message format compatible

---

### Recommended Actions

1. **Update all API calls** to include `conversation_state`
2. **Track both states** in your application
3. **Adjust timeout expectations** (40s instead of 15s)
4. **Implement retry logic** for ZeroGPU queuing
5. **Monitor analytics** via `/refresh_analytics` endpoint

---

## Live Interface

**Hugging Face Space:** https://huggingface.co/spaces/jdesiree/Mimir  
**Direct App URL:** https://jdesiree-mimir.hf.space  
**Model Repository:** https://huggingface.co/jdesiree/Mimir-Phi-3.5  
**Analytics Dataset:** https://huggingface.co/datasets/jdesiree/mimir_analytics  
**Case Study:** https://github.com/Jdesiree112/Technical_Portfolio/tree/main/CaseStudy_Mimir

---

## Support

For issues, questions, or feature requests:
- **Space Discussions**: Comment on the HuggingFace Space page
- **GitHub Issues**: (if repository linked)
- **Documentation**: See architectural docs in Space files tab

---

**Last Updated:** October 2025  
**Architecture Version:** 2.0  
**API Version:** 2.0
