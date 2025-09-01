# Mimir AI Tutoring Application - Complete Issue Analysis & Solutions

## Executive Summary

This analysis covers critical issues affecting the Mimir AI tutoring application, prioritized by impact on user experience and functionality. The application shows promise but requires immediate attention to LangChain deprecation warnings and tool validation errors that compromise core functionality.

---

## Issue Analysis & Solutions

### 1. LangChain Deprecation (HIGH PRIORITY)
**Problem**: LangChain agents are deprecated in favor of LangGraph
**Impact**: Functionality degradation, future incompatibility
**Evidence**: Container log shows deprecation warnings

**Solution - LangGraph Migration**:
```python
# Replace existing agent initialization with LangGraph
from langgraph.prebuilt import ToolExecutor
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated, Union
from langchain_core.agents import AgentAction, AgentFinish

class AgentState(TypedDict):
    input: str
    agent_outcome: Union[AgentAction, AgentFinish, None]
    intermediate_steps: Annotated[list[tuple[AgentAction, str]], operator.add]

# Create the graph
workflow = StateGraph(AgentState)

# Add nodes
workflow.add_node("agent", run_agent)
workflow.add_node("action", execute_tools)

# Define conditional logic
workflow.add_conditional_edges(
    "agent",
    should_continue,
    {
        "continue": "action",
        "end": END
    }
)

workflow.add_edge("action", "agent")
workflow.set_entry_point("agent")

app = workflow.compile()
```

### 2. LLM Runnable Validation Error (HIGH PRIORITY)
**Problem**: `Input should be an instance of Runnable` error
**Impact**: Tool processing failures, degraded user experience
**Evidence**: Container shows 2 validation errors for LLMChain

**Root Cause**: The custom `Qwen25SmallLLM` class doesn't inherit from LangChain's `Runnable` base class

**Solution**:
```python
from langchain_core.runnables import Runnable
from langchain_core.runnables.utils import Input, Output

class Qwen25SmallLLM(Runnable):
    def __init__(self, model_path: str = "Qwen/Qwen2.5-3B-Instruct", use_4bit: bool = True):
        super().__init__()
        # Existing initialization code...
    
    def invoke(self, input: Input, config=None) -> Output:
        # Existing invoke method with proper typing
        return super().invoke(input, config)
    
    @property
    def InputType(self) -> Type[Input]:
        return str
    
    @property 
    def OutputType(self) -> Type[Output]:
        return str
```

### 3. Incorrect Tool Classification (MEDIUM PRIORITY)
**Problem**: Model incorrectly triggers graph tool for warmup message
**Impact**: Unnecessary tool calls, potential performance degradation

**Testing Results Analysis**:

| Test Case | Expected | Actual | Status |
|-----------|----------|--------|---------|
| "hello" | No tools | ✅ No tools | PASS |
| Linear algebra quiz request | Tools needed | ✅ Tools enabled | PASS |

**Solution - Improved Decision Logic**:
```python
def should_use_visualization(self, query: str) -> bool:
    """Enhanced decision logic with explicit exclusions"""
    
    # Explicit exclusions for common non-visual queries
    exclusion_patterns = [
        r'^(hello|hi|hey)\b',
        r'warmup.*test',
        r'(what is|define|explain)\s+\w+\s*(of|the)?',
        r'simple.*question'
    ]
    
    query_lower = query.lower().strip()
    
    # Check exclusions first
    for pattern in exclusion_patterns:
        if re.search(pattern, query_lower):
            return False
    
    # Enhanced positive indicators
    visualization_indicators = [
        'graph', 'chart', 'plot', 'visualiz', 'trend',
        'data', 'statistic', 'compare', 'distribution',
        'function', 'equation', 'relationship'
    ]
    
    decision_query = f"""
    Query: "{query}"
    
    Analyze if this educational query would benefit from visualization.
    
    EXCLUDE if query is:
    - Greetings or casual conversation
    - Simple definitions without data
    - Test/warmup messages
    
    INCLUDE if query involves:
    - Mathematical relationships or functions
    - Data analysis or statistics  
    - Comparisons that benefit from charts
    - Trends or patterns over time
    
    Decision (YES/NO):"""
    
    try:
        response = self.decision_llm.invoke(decision_query)
        decision = response.strip().upper()
        
        # More strict parsing
        if "YES" in decision and "NO" not in decision:
            return True
        return False
        
    except Exception as e:
        logger.error(f"Decision engine error: {e}")
        return False
```

### 4. Deprecated Environment Variables (MEDIUM PRIORITY)
**Problem**: Using deprecated `TRANSFORMERS_CACHE` and `torch_dtype`
**Impact**: Future compatibility issues

**Solution**:
```python
# In app.py, replace:
os.environ['TRANSFORMERS_CACHE'] = '/tmp/huggingface'

# With:
os.environ['HF_HOME'] = '/tmp/huggingface'
# Remove TRANSFORMERS_CACHE entirely

# In model loading, replace:
torch_dtype=torch.bfloat16

# With: 
dtype=torch.bfloat16
```

---

## Testing Recommendations

### Complete Test Suite
```python
test_cases = [
    # Should NOT trigger tools
    ("hello", False),
    ("What is the capital of France?", False), 
    ("Define photosynthesis", False),
    ("warmup test", False),
    
    # SHOULD trigger tools
    ("Create a quiz on linear algebra plotting", True),
    ("Show me trends in climate data", True),
    ("Compare performance across subjects", True),
    ("Graph the function y = mx + b", True)
]

def run_comprehensive_tests():
    agent = get_agent()
    results = []
    
    for query, expected_tools in test_cases:
        actual_tools = agent.should_use_tools(query)
        status = "PASS" if actual_tools == expected_tools else "FAIL"
        results.append((query, expected_tools, actual_tools, status))
        
    return results
```

---

## Implementation Priority

### Phase 1 (Immediate - Critical Functionality)
1. Fix LLM Runnable inheritance issue
2. Complete missing test case analysis
3. Implement improved tool decision logic

### Phase 2 (Short-term - Modernization) 
1. Migrate from LangChain agents to LangGraph
2. Update deprecated environment variables
3. Comprehensive testing suite implementation

### Phase 3 (Long-term - Enhancement)
1. Performance monitoring and optimization
2. Advanced tool decision machine learning
3. User experience improvements

---

## Success Metrics

**Technical Health**:
- Zero validation errors in container logs
- 100% test case accuracy for tool decisions
- Sub-2 second response times

**User Experience**:
- Appropriate tool usage (no unnecessary graphs)
- Consistent educational responses  
- Reliable functionality across all features

---

## Risk Assessment

**High Risk**: LLM validation errors could cause complete tool system failure
**Medium Risk**: Deprecated dependencies may break in future updates
**Low Risk**: Minor tool decision accuracy issues affect user experience but not core functionality

This analysis provides a complete roadmap for resolving Mimir's technical debt while maintaining its educational mission.

## Followup
After implementing changes, the application demonstrated an increase in model load time, from **113.52 seconds** to **97.80 seconds**. Deprecation warnings are no longer present as these issues have been resolved. Also, in this new build, the model correctly decides not to enable the graph tool with the warmup message, which is correct. While loading the application is clearing the conversation history.

**Tool Testing**
**Test Case:** "hello"
**Correct Use:** Query doesn't need tools
*Actual*
  INFO:__main__:Query doesn't need tools - responding normally
**Verdict:** PASS

**Test Case:** "I need help practicing for my upcoming linear algebra test, which is on plotting y = mx + b with provided variables. Can you put together a short practice quiz?"
**Correct Use:** Query requires visualization - enabling tools
*Actual*
  INFO:__main__:Query requires visualization - enabling tools
**Verdict:** PASS

As expected, the model continued to consistently perform well on propperly enabling, or not enabling, the graph tool.

### Further Improvement Needed
While the application has improved, issues remain that need to be addressed. The application runs slowly after a prompt is sent, remaining blank with no visible progress for 25 to 45 seconds, after checking if the graph tool should be enabled.

#### Old Build Container Log
```
/usr/local/lib/python3.10/site-packages/transformers/utils/hub.py:111: FutureWarning: Using `TRANSFORMERS_CACHE` is deprecated and will be removed in v5 of Transformers. Use `HF_HOME` instead.
  warnings.warn(
Environment variables loaded.
Error loading metrics: Expecting value: line 1 column 1 (char 0)
INFO:__main__:==================================================
INFO:__main__:Starting Mimir Application
INFO:__main__:==================================================
INFO:__main__:Loading AI model...
INFO:__main__:Loading model: Qwen/Qwen2.5-1.5B-Instruct (use_4bit=True)

tokenizer_config.json:   0%|          | 0.00/7.30k [00:00<?, ?B/s]
tokenizer_config.json: 100%|██████████| 7.30k/7.30k [00:00<00:00, 25.6MB/s]

vocab.json:   0%|          | 0.00/2.78M [00:00<?, ?B/s]
vocab.json: 100%|██████████| 2.78M/2.78M [00:00<00:00, 60.6MB/s]

merges.txt:   0%|          | 0.00/1.67M [00:00<?, ?B/s]
merges.txt: 100%|██████████| 1.67M/1.67M [00:00<00:00, 34.9MB/s]

tokenizer.json:   0%|          | 0.00/7.03M [00:00<?, ?B/s]
tokenizer.json: 100%|██████████| 7.03M/7.03M [00:00<00:00, 23.6MB/s]

config.json:   0%|          | 0.00/660 [00:00<?, ?B/s]
config.json: 100%|██████████| 660/660 [00:00<00:00, 3.05MB/s]
`torch_dtype` is deprecated! Use `dtype` instead!
WARNING:bitsandbytes.cextension:The 8-bit optimizer is not available on your device, only available on CUDA for now.

model.safetensors:   0%|          | 0.00/3.09G [00:00<?, ?B/s]
model.safetensors:   3%|▎         | 82.4M/3.09G [00:01<00:36, 82.3MB/s]
model.safetensors:   9%|▊         | 269M/3.09G [00:02<00:19, 143MB/s]  
model.safetensors:  17%|█▋        | 529M/3.09G [00:03<00:13, 186MB/s]
model.safetensors:  23%|██▎       | 717M/3.09G [00:04<00:12, 186MB/s]
model.safetensors:  32%|███▏      | 1.00G/3.09G [00:05<00:09, 215MB/s]
model.safetensors:  44%|████▍     | 1.36G/3.09G [00:06<00:06, 259MB/s]
model.safetensors:  56%|█████▌    | 1.74G/3.09G [00:07<00:04, 295MB/s]
model.safetensors:  80%|███████▉  | 2.47G/3.09G [00:08<00:01, 430MB/s]
model.safetensors: 100%|██████████| 3.09G/3.09G [00:08<00:00, 349MB/s]

generation_config.json:   0%|          | 0.00/242 [00:00<?, ?B/s]
generation_config.json: 100%|██████████| 242/242 [00:00<00:00, 1.49MB/s]
INFO:__main__:Model loaded successfully in 113.52 seconds
INFO:__main__:Warming up model...
INFO:__main__:Warming up agent with test query...
INFO:__main__:Tool decision for 'Hello, this is a warmup test....': YES
INFO:__main__:Query requires visualization - enabling tools
/home/user/app/app.py:308: LangChainDeprecationWarning: Please see the migration guide at: https://python.langchain.com/docs/versions/migrating_memory/
  memory = ConversationBufferWindowMemory(
/home/user/app/app.py:321: LangChainDeprecationWarning: LangChain agents will continue to be supported, but it is recommended for new use cases to be built with LangGraph. LangGraph offers a more flexible and full-featured framework for building agents, including support for tool-calling, persistence of state, and human-in-the-loop workflows. For details, refer to the `LangGraph documentation <https://langchain-ai.github.io/langgraph/>`_ as well as guides for `Migrating from AgentExecutor <https://python.langchain.com/docs/how_to/migrate_agent/>`_ and LangGraph's `Pre-built ReAct agent <https://langchain-ai.github.io/langgraph/how-tos/create-react-agent/>`_.
  agent = initialize_agent(
ERROR:__main__:Error in tool processing: 2 validation errors for LLMChain
llm.is-instance[Runnable]
  Input should be an instance of Runnable [type=is_instance_of, input_value=<__main__.Qwen25SmallLLM object at 0x7f9ca0aabf70>, input_type=Qwen25SmallLLM]
    For further information visit https://errors.pydantic.dev/2.11/v/is_instance_of
llm.is-instance[Runnable]
  Input should be an instance of Runnable [type=is_instance_of, input_value=<__main__.Qwen25SmallLLM object at 0x7f9ca0aabf70>, input_type=Qwen25SmallLLM]
    For further information visit https://errors.pydantic.dev/2.11/v/is_instance_of
INFO:__main__:Agent warmup completed successfully! Test response length: 633 chars
INFO:httpx:HTTP Request: GET https://api.gradio.app/pkg-version "HTTP/1.1 200 OK"
* Running on local URL:  http://0.0.0.0:7860, with SSR ⚡ (experimental, to disable set `ssr_mode=False` in `launch()`)
INFO:httpx:HTTP Request: GET http://localhost:7860/gradio_api/startup-events "HTTP/1.1 200 OK"
INFO:httpx:HTTP Request: HEAD http://0.0.0.0:7861/ "HTTP/1.1 200 OK"
INFO:httpx:HTTP Request: HEAD http://localhost:7860/ "HTTP/1.1 200 OK"
/usr/local/lib/python3.10/site-packages/gradio/blocks.py:2884: UserWarning: Setting share=True is not supported on Hugging Face Spaces
  warnings.warn(
* To create a public link, set `share=True` in `launch()`.
INFO:httpx:HTTP Request: GET http://0.0.0.0:7861/?__theme=system "HTTP/1.1 200 OK"
INFO:httpx:HTTP Request: GET http://0.0.0.0:7861/?__theme=light "HTTP/1.1 304 Not Modified"
INFO:__main__:Metrics interaction logged successfully
```
#### New Build Container Log
```
Environment variables loaded.
Error loading metrics: Expecting value: line 1 column 1 (char 0)
INFO:__main__:==================================================
INFO:__main__:Starting Mimir Application
INFO:__main__:==================================================
INFO:__main__:Loading AI model...
INFO:__main__:Loading model: Qwen/Qwen2.5-1.5B-Instruct (use_4bit=True)

tokenizer_config.json:   0%|          | 0.00/7.30k [00:00<?, ?B/s]
tokenizer_config.json: 100%|██████████| 7.30k/7.30k [00:00<00:00, 33.4MB/s]

vocab.json:   0%|          | 0.00/2.78M [00:00<?, ?B/s]
vocab.json: 100%|██████████| 2.78M/2.78M [00:00<00:00, 76.4MB/s]

merges.txt:   0%|          | 0.00/1.67M [00:00<?, ?B/s]
merges.txt: 100%|██████████| 1.67M/1.67M [00:00<00:00, 25.3MB/s]

tokenizer.json:   0%|          | 0.00/7.03M [00:00<?, ?B/s]
tokenizer.json: 100%|██████████| 7.03M/7.03M [00:00<00:00, 36.6MB/s]

config.json:   0%|          | 0.00/660 [00:00<?, ?B/s]
config.json: 100%|██████████| 660/660 [00:00<00:00, 5.01MB/s]
WARNING:bitsandbytes.cextension:The 8-bit optimizer is not available on your device, only available on CUDA for now.

model.safetensors:   0%|          | 0.00/3.09G [00:00<?, ?B/s]
model.safetensors:   2%|▏         | 48.7M/3.09G [00:01<01:02, 48.6MB/s]
model.safetensors:   8%|▊         | 246M/3.09G [00:02<00:22, 129MB/s]  
model.safetensors:  15%|█▍        | 451M/3.09G [00:03<00:16, 160MB/s]
model.safetensors:  22%|██▏       | 689M/3.09G [00:04<00:12, 185MB/s]
model.safetensors:  36%|███▌      | 1.11G/3.09G [00:05<00:07, 263MB/s]
model.safetensors:  52%|█████▏    | 1.61G/3.09G [00:06<00:04, 340MB/s]
model.safetensors:  74%|███████▎  | 2.27G/3.09G [00:07<00:01, 441MB/s]
model.safetensors: 100%|██████████| 3.09G/3.09G [00:08<00:00, 385MB/s]

generation_config.json:   0%|          | 0.00/242 [00:00<?, ?B/s]
generation_config.json: 100%|██████████| 242/242 [00:00<00:00, 1.42MB/s]
INFO:__main__:Model loaded successfully in 97.80 seconds
INFO:__main__:Warming up model...
INFO:__main__:Warming up agent with test query...
INFO:__main__:Query doesn't need tools - responding normally
INFO:__main__:Agent warmup completed successfully! Test response length: 34 chars
INFO:httpx:HTTP Request: GET https://api.gradio.app/pkg-version "HTTP/1.1 200 OK"
* Running on local URL:  http://0.0.0.0:7860, with SSR ⚡ (experimental, to disable set `ssr_mode=False` in `launch()`)
INFO:httpx:HTTP Request: GET http://localhost:7860/gradio_api/startup-events "HTTP/1.1 200 OK"
INFO:httpx:HTTP Request: HEAD http://0.0.0.0:7861/ "HTTP/1.1 200 OK"
INFO:httpx:HTTP Request: HEAD http://localhost:7860/ "HTTP/1.1 200 OK"
/usr/local/lib/python3.10/site-packages/gradio/blocks.py:2884: UserWarning: Setting share=True is not supported on Hugging Face Spaces
  warnings.warn(
* To create a public link, set `share=True` in `launch()`.
```
