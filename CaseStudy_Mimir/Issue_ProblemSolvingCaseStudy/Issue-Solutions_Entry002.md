# Metrics Recording
## Overview
Mimir has demonstrated critical performance and interface failures when processing user prompts. Initial investigation revealed a multi-layered problem: severe response delays exceeding 9 minutes, complete UI breakdown where user messages fail to display in the chatbot interface, and streaming pipeline failures resulting in empty response generation. I hypothesized that the problems originated from either pipeline bottlenecks or interface configuration issues. To systematically diagnose these failures, I implemented comprehensive metrics logging to capture elapsed time for each pipeline step.

The metrics evaluation revealed that the primary issue lies with fundamental model-environment incompatibility on the target infrastructure. Initially operating on a 2 vCPU system with 16 GB RAM, despite implementing quantization to support Qwen2.5-3B-Instruct, this solution proved ineffective due to the model's computational requirements far exceeding available resources. The quantization overhead itself was consuming significant processing time without delivering expected performance gains. Concurrently, interface failures manifested as user messages not populating in the Gradio chatbot component and the streaming pipeline yielding empty content repeatedly, causing the `smart_truncate` function to process minimal strings in an endless loop.

To resolve these interconnected issues, I migrated the deployment environment to ZeroGPU to leverage hardware acceleration while maintaining cost efficiency. This infrastructure change necessitated updating the model selection criteria to optimize for GPU-accelerated inference and implementing 4-bit quantization using bitsandbytes to maximize VRAM efficiency. Additionally, I replaced the custom streaming implementation with TextIteratorStreamer to eliminate the threading complications that were causing UI update failures and generator chain breaks in the response pipeline.

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

The application experiences catastrophic latency in all model interaction steps, with the 578-second response time representing a fundamental mismatch between the model's computational demands and the available processing capability. This extreme delay, combined with interface failures where user messages don't display and responses fail to generate, indicates multiple system failures cascading from the core model incompatibility. Interface operations have minimal time impact, confirming they are not the root cause. Tool-related functions have negligible overhead, indicating the LangGraph workflow itself is efficient when not constrained by model performance.

### Model Selection

Based on Mimir's educational assistant requirements, any replacement model must meet these critical specifications optimized for GPU acceleration:

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

##### **6. GPU Performance Constraints**
- CUDA compatibility for hardware acceleration
- Efficient VRAM utilization with quantization support
- Model loading under 60 seconds with cold start overhead
- 600-token response generation under 15 seconds
- Compatible with ZeroGPU's ephemeral allocation model

##### **7. Library Compatibility**
- `AutoModelForCausalLM.from_pretrained()` support
- `BitsAndBytesConfig` quantization integration
- `torch.bfloat16` optimization for GPU inference
- `device_map="auto"` configuration for multi-GPU setups

#### **Quality Requirements**

##### **8. Response Generation**
- Coherent 400-600 token educational explanations
- Consistent output formatting
- Reliable decision-making for tool usage
- Stable performance under GPU memory constraints

This specification ensures seamless integration with Mimir's existing architecture while leveraging GPU acceleration capabilities. I began researching first by running a research query through Gemini's deep research function to gather leads on specific models that may serve well for Mimir. From this, the following top candidates were researched, with my findings detailed below:

### Small Language Models Comparison for ZeroGPU Deployment
*NVIDIA H200 GPU, 70GB VRAM Platform*

| Model | Parameters | VRAM Usage (4-bit) | Performance (est. t/s) | Quantization Efficiency | **Pass/Fail** |
|-------|------------|-------------------|----------------------|----------------------|---------------|
| **meta-llama/Meta-Llama-3-8B-Instruct** | 8B | ~4.5 GB | 35-45 | Excellent (75% reduction) | **PASS** |
| **microsoft/Phi-3-mini-4k-instruct** | 3.8B | ~2.2 GB | 50+ | Excellent (71% reduction) | **PASS** |
| **mistralai/Mistral-7B-Instruct-v0.3** | ~7B | ~4.0 GB | 40-50 | Good (70% reduction) | **PASS** |

#### Assessment Criteria for ZeroGPU Deployment

##### Pass Requirements:
- Efficient VRAM utilization under quantization
- Fast cold start performance for ephemeral GPU allocation
- Stable inference under ZeroGPU's execution time constraints
- Compatibility with bitsandbytes optimization

##### Detailed Analysis:

**Llama-3-8B-Instruct - PASS**
- Excellent performance with GPU acceleration
- Efficient quantization reduces VRAM to manageable levels
- Strong educational capabilities with robust instruction following
- Suitable for ZeroGPU's dynamic allocation model

**Phi-3-mini-4k-instruct - PASS** 
- Optimal resource efficiency (3.1% VRAM usage on H200)
- Fastest inference speed with excellent quantization benefits
- Designed specifically for memory-constrained and edge environments
- Superior cold start performance for ephemeral GPU contexts

**Mistral-7B-Instruct-v0.3 - PASS**
- Good performance balance with moderate VRAM usage
- Strong educational reasoning capabilities
- Large context window (32K tokens) beneficial for complex prompts
- Reliable quantization performance

#### Selection
**Phi-3-mini-4k-instruct** remains the optimal choice due to its superior resource efficiency, fastest cold start times critical for ZeroGPU's ephemeral model, and excellent quantization compatibility. The model's design specifically targets scenarios requiring efficient GPU utilization while maintaining high-quality output.

### Educated Updates and Implementations
## Overview
1. Replaced the previous model with Phi-3-mini-4k-instruct
2. Migrated to ZeroGPU infrastructure for hardware acceleration
3. Implemented 4-bit quantization using bitsandbytes
4. Replaced custom streaming implementation with TextIteratorStreamer
5. Removed Fallback Model

These changes aim to resolve the model incompatibility issue, eliminate interface display failures, and optimize for GPU-accelerated performance. The specific rationale for each change addresses critical performance bottlenecks and UI failures identified through metrics analysis:

**Infrastructure Migration Rationale**: Moving from CPU-only to ZeroGPU infrastructure fundamentally resolves multiple interconnected compatibility issues. The NVIDIA H200 GPU provides massive parallel processing capability that transforms the 578-second response time into sub-15-second generation. Critically, this migration also resolves the bitsandbytes incompatibility that was creating additional performance penalties on CPU. Bitsandbytes remains in alpha phase for CPU environments with significant performance degradation, where quantization operations actually slow inference rather than accelerating it. Alternative CPU quantization libraries were explored but proved ineffective due to heavy initial memory overhead during model loading that caused application crashes within the 16GB RAM constraint. The ZeroGPU environment enables bitsandbytes to function as designed, providing 71% VRAM reduction through efficient CUDA kernel operations while maintaining inference speed. This change enables proper model performance while maintaining cost efficiency through dynamic GPU allocation.

**Model Replacement Rationale**: Phi-3-mini-4k-instruct was specifically optimized by Microsoft for efficient inference scenarios, including GPU acceleration. Its architecture is designed for fast cold starts and minimal VRAM overhead, making it ideal for ZeroGPU's ephemeral allocation model where models must load quickly when GPUs are dynamically assigned.

**Quantization Implementation Rationale**: With GPU infrastructure, bitsandbytes 4-bit quantization becomes highly effective, reducing VRAM usage from ~7.6GB to ~2.2GB (71% reduction) while maintaining model quality. This optimization accelerates cold start times and maximizes efficiency within ZeroGPU's resource allocation constraints.

**Streaming Implementation Replacement Rationale**: The custom streaming implementation was causing cascading failures in the UI pipeline through complex threading interactions and manual generator management. Thread synchronization issues created timing problems that prevented user messages from displaying in the Gradio interface while breaking the generator chain. Replacing the custom implementation with TextIteratorStreamer provides a battle-tested, library-maintained solution that handles thread management automatically and integrates seamlessly with the transformers generation pipeline, eliminating these threading complications and ensuring reliable message display.

**Fallback Model Removal Rationale**: With optimized GPU infrastructure and a properly selected model, the fallback system becomes unnecessary overhead that complicates the deployment architecture without providing meaningful benefits.

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
