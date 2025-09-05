# Metrics Recording
## Overview
Mimir has demonstrated prolonged response delays when processing user prompts. I hypothesized that the problem originated from either a pipeline bottleneck or interface issues. To address this, I implemented comprehensive metrics logging to capture elapsed time for each pipeline step. Based on the metrics evaluation, I determined that the issue lies not with the LangChain pipeline or interface, but with model-environment incompatibility on the target CPU. Mimir operates on a 2 vCPU system with 16 GB RAM. Despite implementing quantization in app.py to support Qwen2.5-3B-Instruct, a model designed for Mimir's use cases, this solution proved ineffective. To resolve this issue, I selected a replacement model optimized for CPU inference.

### Initial Metrics Evaluation

| Step | Average Time (seconds) |
| :- | :-: |
| Model Load time | 120.69 |
| Init and LangGraph Workflow Setup | 120.70 |
| Tool Decision Time | 0.0002 |
| **LLM Invoke** | **578.79** |
| **Call Model (no tools)** | **578.79** |
| **Complete Chat Time** | **578.80** |
| Create Interface | 0.20 |

The application experiences significant latency in all model interaction steps, indicating hardware-software incompatibility. Interface operations have minimal time impact, confirming they are not the root cause. Tool-related functions have negligible overhead.

### Model Selection

Based on Mimir's educational assistant requirements, any replacement model must meet these critical specifications:

#### **Core Compatibility Requirements**

##### **1. Chat Template Support**
- Built-in `tokenizer.apply_chat_template()` support
- System, user, and assistant message role handling
- No manual prompt formatting requirements

##### **2. System Prompt Processing**
- Handle 2000+ character educational system prompts
- Follow complex multi-section instructions
- Maintain consistent behavior across prompt complexity

##### **3. JSON Generation for Tool Calling**
- Generate valid JSON with specific schema for graph tool integration
- Parse tool call prompts and respond with `TOOL_CALL: Create_Graph_Tool` format
- Avoid malformed JSON or field hallucination

##### **4. Educational Instruction Following**
- Understand academic integrity concepts
- Provide step-by-step guidance without direct answers
- Maintain appropriate educational tone for high school students

#### **Technical Requirements**

##### **5. Framework Integration**
- LangChain Runnable interface compatibility
- Custom LLM class inheritance support
- Message object handling (HumanMessage, AIMessage, SystemMessage, ToolMessage)

##### **6. Performance Constraints**
- CPU-only operation on 16GB RAM systems
- Model loading under 30 seconds
- 600-token response generation under 30 seconds
- Maximum 13GB model size

##### **7. Library Compatibility**
- `AutoModelForCausalLM.from_pretrained()` support
- `dtype=torch.float16` memory optimization
- `device_map="cpu"` configuration

#### **Quality Requirements**

##### **8. Response Generation**
- Coherent 400-600 token educational explanations
- Consistent output formatting
- Reliable decision-making for tool usage

This specification ensures seamless integration with Mimir's existing architecture without compromising functionality. I bgan researching first by running a research query through Gemini's deep researhc function to gather leads on specific models that may serve well for Mimir. From this, I the following top candidates were researched, with my finding detailed below:

### Small Language Models Comparison for Hugging Face Spaces Deployment
*2 vCPU, 16GB RAM Platform*

| Model | Parameters | File Size (Q4_K_M) | RAM Usage | Performance (est. t/s) | Memory Pressure | **Pass/Fail** |
|-------|------------|-------------------|-----------|----------------------|----------------|---------------|
| **meta-llama/Meta-Llama-3-8B-Instruct** | 8B | 4.92 GB | ~10 GB | 11-13 | High (62.5% RAM usage) | **FAIL** |
| **microsoft/Phi-3-mini-4k-instruct** | 3.8B | ~2.2 GB | ~6 GB | 20+ | Low (37.5% RAM usage) | **PASS** |
| **mistralai/Mistral-7B-Instruct-v0.3** | ~7B | 4.37 GB | ~7 GB | 18-20 | Medium (43.8% RAM usage) | **PASS** |

#### Assessment Criteria for Hugging Face Spaces

##### Pass Requirements:
- Model + inference overhead must fit comfortably within 16GB RAM
- Leave sufficient memory headroom for stable operation
- Avoid memory swapping that could cause timeouts or crashes
- Maintain reasonable inference speed on limited CPU resources

##### Detailed Analysis:

**Llama-3-8B-Instruct - FAIL**
- Consumes 62.5% of available RAM (~10GB total usage)
- High risk of memory pressure and swapping
- Minimal headroom for OS and application overhead
- Could cause instability in constrained Spaces environment

**Phi-3-mini-4k-instruct - PASS** 
- Excellent resource efficiency (37.5% RAM usage)
- Fastest inference speed despite smallest size
- Large memory headroom ensures stability
- Designed specifically for memory-constrained environments

**Mistral-7B-Instruct-v0.3 - PASS**
- Moderate resource usage (43.8% RAM usage)
- Good performance with reasonable memory footprint
- Adequate headroom for stable operation
- Largest context window (32K tokens) as bonus feature

#### Selection
**Phi-3-mini-4k-instruct** is the optimal choice due to its superior resource efficiency and reliability in constrained environments. Before implmenting any changes, I researched in detail the architecture and available documentation of Phi-3-mini-4k-instruct.

### Educated Updates and Implmentations
## Overview
1. Replaced the previous model with Phi-3-mini-4k-instruct
2. Removed bulky streaming script for TextIteratorStreamer
3. Removed Fallback Model
4. Removed quantumization

This changes aim to resolve the model incompatability issue, improove user experience, and reduce excess from the application. The streaming script that was previously implmented presented issues, so it has been replaced with a package that delivers the desired result with much less weight. 

Due to memory constraints, quantumization has been removed.

### Followup Metrics Evaluation

| Step | Average Time (seconds) |
| :- | :-: |
| Model Load time | 39.86 |
| Init and LangGraph Workflow Setup | 39.86 |
| Tool Decision Time | 0.0002 |
| **LLM Invoke** | **** |
| **Call Model (no tools)** | **** |
| **Complete Chat Time** | **** |
| Create Interface |  |

