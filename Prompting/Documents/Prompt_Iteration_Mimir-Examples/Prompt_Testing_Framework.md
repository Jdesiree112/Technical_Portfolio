# Prompt Testing Framework

## Overview

The Prompt Testing Framework provides automated evaluation of prompt effectiveness across Mimir's multi-agent orchestration pipeline. This comprehensive testing application was integrated into Mimir to systematically measure and analyze the performance of individual agents as they operate within the full production pipeline.

The framework captures granular metrics at each step of the orchestration process, including:
- **Agent activation patterns**: Which agents are triggered for different types of queries
- **Token utilization**: Input and output tokens for each model invocation across the pipeline
- **Execution timing**: Precise timing measurements for each agent and the overall pipeline
- **Resource consumption**: GPU memory usage peaks during model inference
- **Response quality**: Readability scores, completeness metrics, and educational effectiveness indicators
- **Decision pathways**: The logical flow through routing agents and conditional thinking agent activation

By mirroring the exact orchestration flow implemented in `app.py`, the testing framework ensures that evaluation results reflect real-world system behavior. This design allows for iterative prompt engineering with confidence that improvements tested in the framework will translate directly to production performance.

The framework supports both bulk testing via CSV upload and manual entry of individual prompts, generating comprehensive CSV exports with approximately 110 columns of metrics. This data enables detailed analysis of prompt performance, identification of optimization opportunities, and validation of system changes before deployment.

## Architecture

### System Components

The Prompt Testing Framework is built on a modular architecture that replicates Mimir's production pipeline while adding comprehensive instrumentation for metrics collection.

#### Core Agent Stack

**1. Tool Decision Agent**
- **Purpose**: Determines whether the user query requires external tool usage (calculators, graphing, web search)
- **Input**: User prompt + conversation history
- **Output**: Binary decision (tools required: yes/no) with reasoning
- **Metrics Captured**: Input tokens, output tokens, inference time, GPU memory peak, decision result

**2. Prompt Routing Agents (Agent 1-4)**
- **Purpose**: Four-stage classification system that determines which specialized prompts and thinking agents should be activated
- **Agent 1**: Identifies if the query requires mathematical thinking
- **Agent 2**: Determines if question/answer design prompts are needed
- **Agent 3**: Assesses whether reasoning/logic thinking is required
- **Agent 4**: Evaluates if vague input clarification is needed
- **Processing**: Each agent runs independently and returns a boolean decision
- **Metrics Captured**: Per-agent input templates, token counts, outputs, decisions, timing, and GPU usage

**3. Thinking Agents (Conditional Activation)**
- **Math Thinking Agent**: Activated for mathematical problem-solving, formula derivation, and calculations
- **Question Answer Design Agent**: Engaged for creating practice questions, assessments, and structured learning materials
- **Reasoning Thinking Agent**: Invoked for logical reasoning, critical thinking, and complex problem analysis
- **Activation Logic**: Only activated when corresponding routing agent returns True
- **Metrics Captured**: Input data structure, output content, token counts, processing time, GPU memory

**4. Response Agent**
- **Purpose**: Generates the final user-facing response using Llama-3.2-3B model
- **Input**: Structured input_data dictionary containing user prompt, tool results, thinking agent outputs, and conversation context
- **Output**: Complete natural language response
- **Post-processing**: Applies formatting rules, LaTeX rendering preparation, and safety checks
- **Metrics Captured**: Template construction, total input tokens, generated tokens, inference time, GPU usage

### Data Flow Pipeline

```
User Prompt Input
       ↓
[Tool Decision Agent] → Regex Checks (pattern matching for tool triggers)
       ↓
[Routing Agent 1] → Math Required?
[Routing Agent 2] → QA Design Required?
[Routing Agent 3] → Reasoning Required?
[Routing Agent 4] → Vague Input?
       ↓
[Conditional Thinking Agents] → Activated based on routing decisions
       ↓
[Response Agent] → Synthesis of all inputs into final response
       ↓
[Post-Processor] → Formatting and safety checks
       ↓
Final Output + Comprehensive Metrics
```

### Metrics Collection System

The framework captures approximately 110 distinct metrics organized into the following categories:

**Input Metrics**
- User prompt (tokens, characters, words)
- Conversation history length and token count
- Timestamp and prompt index

**Agent Performance Metrics** (per agent)
- Input template construction
- Input token count
- Raw output text
- Output token count
- Decision/classification result
- Execution time (seconds)
- GPU memory peak (MB)

**Pipeline-Level Metrics**
- Total models activated
- Total pipeline execution time
- Cumulative input tokens
- Cumulative output tokens
- Peak GPU memory usage
- Tool usage statistics

**Quality Metrics**
- Flesch Reading Ease score
- Flesch-Kincaid Grade Level
- Automated Readability Index (ARI)
- Dale-Chall Readability Score
- Average word length
- Average sentence length
- Completeness score (0-1)
- Question answered boolean
- Explanation quality indicators

### State Management Integration

The framework integrates with Mimir's `GlobalStateManager` and `LogicalExpressions` modules to:
- Track conversation context across multi-turn interactions
- Maintain state for conditional agent activation
- Apply regex-based decision shortcuts that bypass model inference when appropriate
- Manage prompt template construction with dynamic context injection

### Technical Implementation

**Model Management**
- Utilizes shared model instances via `model_manager.get_model()` to avoid redundant loading
- Supports ZeroGPU integration for efficient GPU allocation in serverless environments
- Implements torch-based GPU memory tracking for resource monitoring

**Token Counting**
- Primary: `tiktoken` library for accurate OpenAI-compatible token counting
- Fallback: Character-based estimation when tiktoken unavailable
- Captures tokens at every transformation step in the pipeline

**Error Handling**
- Graceful degradation when optional dependencies unavailable
- Per-prompt error capture without aborting batch processing
- Detailed error logging with stack traces for debugging

**Export Format**
- CSV output with timestamped filenames
- Self-documenting column headers
- Compatible with standard data analysis tools (Excel, Pandas, R)

### Interface Design

Built with Gradio, the testing interface provides:

**Input Methods**
1. **CSV Upload**: Bulk testing with structured input files
2. **Manual Entry**: Line-separated prompts for quick testing

**Real-time Feedback**
- Progress status updates during batch processing
- Summary statistics upon completion
- Downloadable CSV with complete metrics

**Configuration Options**
- Custom test naming for organized result tracking
- Batch size handling for memory management
- Debug mode for detailed logging

### Validation and Quality Assurance

**Pipeline Parity**
- Tool decision uses `decide()` method matching production implementation
- Response agent receives `input_data` dict structure identical to `app.py`
- Thinking agents invoke `process()` method with correct parameters
- Post-processing applies the same transformations as production system

**Continuous Verification**
- Automated checks confirm agent initialization
- Dependency availability logged at startup
- Model loading failures caught with informative errors
- Results validation ensures metric completeness

### Use Cases

**1. Prompt Engineering Optimization**
- A/B testing of system prompts across agent types
- Evaluation of prompt template modifications
- Impact assessment of instruction changes

**2. Performance Benchmarking**
- Token usage analysis for cost optimization
- Inference time profiling for latency reduction
- GPU memory optimization studies

**3. Quality Assurance**
- Regression testing for system updates
- Validation of multi-agent coordination logic
- Response quality monitoring across prompt categories

**4. Research and Development**
- Analysis of agent activation patterns
- Identification of redundant processing steps
- Data collection for machine learning improvements

### Extensibility

The framework is designed for easy extension:
- **New Agents**: Add metrics capture in the pipeline processing loop
- **Custom Metrics**: Extend CSV_COLUMNS and calculation functions
- **Alternative Models**: Swap model implementations while preserving metric collection
- **Enhanced Analysis**: Output format compatible with downstream analytics pipelines

---

## Getting Started

### Prerequisites
- Mimir application codebase (`agents.py`, `model_manager.py`, `state_manager.py`, `prompt_library.py`)
- Python 3.8+
- Dependencies: PyTorch, Gradio, transformers, tiktoken, textstat

### Running Tests

**Manual Entry Mode:**
1. Launch the Gradio interface
2. Enter prompts (one per line) in the text area
3. Provide a test name for result identification
4. Click "Run Full Pipeline Test"
5. Download the generated CSV with complete metrics

**CSV Upload Mode:**
1. Prepare a CSV with one prompt per row (with or without header)
2. Select "CSV Upload" mode
3. Upload your file
4. Provide a test name
5. Process and download results

### Interpreting Results

The output CSV contains comprehensive metrics for each prompt tested. Key columns to examine:

- `tool_decision_result`: Whether tools were activated
- `agent[1-4]_decision`: Which specialized processing was triggered
- `models_activated_count`: Total number of model invocations
- `total_pipeline_time_seconds`: End-to-end latency
- `total_input_tokens` / `total_output_tokens`: Resource consumption
- `completeness_score`: Response quality indicator
- `flesch_reading_ease`: Readability for target audience

Summary statistics provide aggregated insights across the full test batch, enabling identification of trends and outliers.
