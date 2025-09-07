# Evaluation Framework

## Document Summary

This document demonstrates my comprehensive approach to systematic evaluation and performance optimization in AI applications. It presents my real-world methodologies used in developing Mimir, an AI tutoring application, showcasing iterative testing, metrics-driven analysis, and quantitative assessment workflows. The document also includes my capability to implement automated evaluation frameworks using APIs like OpenAI Evals, though this approach has not been applied to Mimir development.

## Evaluation Methods Overview

My approach to AI application quality assurance employs multiple complementary methodologies to ensure robust performance and user experience:

• **Quantitative Performance Metrics**: Systematic logging and measurement of response times, resource utilization, and throughput across all system components to identify bottlenecks and validate optimizations

• **[Manual Prompt Sampling & Quality Assessment](CaseStudy_Mimir/evaluation/Benchmark_Test_Cases.md)**: Iterative testing using structured benchmark prompts to evaluate educational effectiveness, academic integrity compliance, and subject-specific functionality across diverse use cases

• **Before/After Comparative Analysis**: Structured evaluation of user experience improvements through systematic problem identification, solution implementation, and measurable outcome validation

• **Real-Time Debug Infrastructure**: Comprehensive state monitoring and logging systems that enable continuous evaluation of application behavior and rapid diagnosis of issues during development and production

• **Hypothesis-Driven Root Cause Investigation**: Systematic analysis framework that applies structured hypothesis testing to isolate performance issues and guide targeted optimization efforts

• **Automated Evaluation Capabilities**: Proficiency in implementing large-scale prompt and model evaluation frameworks using APIs like OpenAI Evals for comprehensive testing scenarios requiring extensive validation

• **User Experience Impact Assessment**: Evaluation of perceived performance, educational effectiveness, and feature reliability to ensure technical improvements translate to meaningful user benefits

This multi-faceted evaluation strategy ensures both technical performance optimization and user experience quality while providing clear, quantifiable evidence of improvement effectiveness.

## Mimir Evaluation Methodology

### Performance Metrics Recording
I implement comprehensive metrics logging to systematically diagnose performance bottlenecks and track optimization progress. This approach enabled me to identify and resolve critical issues that were causing 9+ minute response delays.

#### Initial Metrics Evaluation
My metrics framework captures elapsed time for each pipeline component, providing quantitative evidence for optimization decisions:

| Step | Average Time (seconds) |
| :- | :-: |
| Model Load time | 120.69 |
| Init and LangGraph Workflow Setup | 120.70 |
| Tool Decision Time | 0.0002 |
| **LLM Invoke** | **578.79** |
| **Call Model (no tools)** | **578.79** |
| **Complete Chat Time** | **578.80** |
| Create Interface | 0.20 |

This systematic measurement revealed that the 578-second response time represented a fundamental model-infrastructure mismatch, while interface operations had minimal impact, confirming they weren't the root cause.

#### Post-Optimization Results
After implementing comprehensive infrastructure and architectural changes, I achieved dramatic performance improvements:

**Major Changes Implemented:**
1. **Infrastructure Migration**: Switched from 2 vCPU with 16 GB RAM to ZeroGPU with NVIDIA H200 GPU hardware acceleration
2. **Model Optimization**: Replaced Qwen2.5-3B-Instruct with Microsoft Phi-3-mini-4k-instruct, specifically optimized for efficient GPU inference
3. **LangGraph Architecture Refinement**: Enhanced conversation handling and state management through improved LangGraph workflow design
4. **Gradio State Management**: Implemented proper gr.State() handling to eliminate UI synchronization issues and message persistence problems

**Performance Results:**

| Step | Previous (seconds) | Optimized (seconds) | Improvement |
| :- | :-: | :-: | :-: |
| Init and LangGraph Workflow Setup | 120.70 | 1.02 | **99.2% faster** |
| Tool Decision Time | 0.0002 | 0.0001 | Maintained |
| **LLM Invoke (Warmup)** | **578.79** | **17.64** | **97.0% faster** |
| **LLM Stream (User Interaction)** | **578.79** | **32.25** | **94.4% faster** |
| **Total Query Processing** | **578.80** | **34.19** | **94.1% faster** |
| Create Interface | 0.20 | 0.21 | Maintained |

**Infrastructure Impact Analysis:**
- **Model Loading**: Cold start optimization reduced from 120+ seconds to ~1 second through ZeroGPU's optimized model caching
- **GPU Acceleration**: 4-bit quantization with bitsandbytes achieved 71% VRAM reduction (7.6GB → 2.2GB) while maintaining inference quality
- **Response Generation**: Real user interactions now complete in 30-35 seconds vs. previous 9+ minutes, representing a 94% improvement in user experience

**Quantifiable Improvements:**
- **Startup Time**: 97% reduction in initialization overhead
- **Response Latency**: 94% reduction in query processing time  
- **Memory Efficiency**: 71% VRAM reduction through optimized quantization
- **Error Rate**: Eliminated validation and deprecation errors (100% resolution)

### Manual Prompt Sampling & Quality Assessment
I implement systematic manual testing using structured benchmark prompts to evaluate educational effectiveness and application functionality across diverse academic scenarios. This methodology provides qualitative validation of educational integrity and subject-specific performance that complements quantitative metrics.

**Testing Framework:**
- **Subject Area Coverage**: Math (LaTeX formatting, academic integrity), Science (graph tool utilization), English/Language Arts (grammar, vocabulary, reading comprehension), Research (information literacy), and Test Preparation (comprehensive strategy development)
- **Academic Integrity Validation**: High-risk test cases specifically designed to evaluate whether the system appropriately guides rather than provides direct homework solutions
- **Technical Implementation Testing**: Verification of LaTeX rendering, graph generation tool integration, and proper mathematical notation formatting
- **Educational Methodology Assessment**: Evaluation of teaching approach, age-appropriate communication, and learning scaffolding effectiveness

**Sample Benchmark Categories:**

**Math Mode Assessment**:
- Basic concept explanation: "What does cos, tan and sin do?" - Evaluates LaTeX implementation and conceptual vs. procedural focus
- Direct problem requests: "Solve 3x - y = 7 and 2x + 3y = 1" - Tests academic integrity adherence and teaching methodology over answer provision
- Mathematical notation: "What is summation?" - Assesses comprehensive concept explanation with proper LaTeX formatting

**Science Mode Assessment**:
- Data interpretation: "I need help understanding how to interpret graphs in the science portion of my exam" - Validates graph tool utilization and educational scaffolding

**English/Language Arts Assessment**:
- Test preparation: "I want to practice my English comprehension for LSAT" - Evaluates understanding of test-specific requirements and appropriate practice methodology
- Grammar mechanics: Multi-topic grammar requests - Tests comprehensive coverage and clear explanations with examples

**Evaluation Metrics**:
- **API Call Success Rate**: Technical execution reliability across different prompt types
- **Response Time Consistency**: Execution time variation across subject areas and complexity levels
- **Educational Quality Score**: Manual assessment based on teaching effectiveness, academic integrity compliance, and age-appropriate communication
- **Feature Integration Success**: Proper utilization of LaTeX rendering, graph generation tools, and other technical capabilities

This iterative testing approach enables continuous validation of educational effectiveness while identifying areas requiring prompt optimization or system enhancement.

### User Experience Analysis
My evaluation process includes systematic before/after analysis of interface issues:

**Core Problems Identified:**
- Messages disappearing from conversation history
- User messages delayed until after AI response
- No feedback during long processing times (30+ seconds)
- LaTeX rendering issues and inconsistent formatting

**Solutions Implemented with Measurable Outcomes:**
- **Immediate User Feedback**: Three-stage response flow (add_user_message → add_thinking_indicator → generate_response)
- **Visual Loading Indicators**: Context-aware animated messages with CSS-based animations
- **State Management Fixes**: Proper Gradio state management eliminating message disappearance
- **Native LaTeX Integration**: Eliminated MathJax conflicts through proper delimiter configuration

### Debug Infrastructure and State Monitoring
I implement comprehensive debugging systems for continuous evaluation and real-time state inspection. This infrastructure enables systematic tracking of conversation flow issues and validation of state management fixes:

```python
def debug_state(conversation_state, event_name="", force_debug=False):
    """Debug function to inspect current conversation state"""
    if not (DEBUG_STATE or force_debug):
        return conversation_state
    
    timestamp = datetime.now().strftime("%H:%M:%S")
    logger.info(f"[{timestamp}] DEBUG STATE - {event_name}")
    logger.info(f"Total messages: {len(conversation_state)}")
    
    for i, msg in enumerate(conversation_state):
        role = msg["role"]
        content_preview = msg["content"][:100] + "..." if len(msg["content"]) > 100 else msg["content"]
        logger.info(f" {i+1}. {role}: {content_preview}")
    
    # Log to file for later analysis
    if DEBUG_STATE:
        debug_log_file = "debug_state.log"
        with open(debug_log_file, "a", encoding="utf-8") as f:
            f.write(f"\n=== {timestamp} - {event_name} ===\n")
            f.write(f"Total messages: {len(conversation_state)}\n")
            for i, msg in enumerate(conversation_state):
                f.write(f"{i+1}. {msg['role']}: {msg['content'][:200]}...\n")
            f.write("=" * 40 + "\n")
    
    return conversation_state
```

**Key Features:**
- **Conditional Debugging**: Environment-controlled debug mode prevents performance overhead in production
- **Real-time Inspection**: Immediate logging of conversation state at key interaction points
- **Persistent Analysis**: File-based logging enables post-hoc analysis of conversation patterns
- **Content Preview**: Truncated message display balances detail with readability
- **Event Correlation**: Timestamping supports correlation with performance metrics

This debugging infrastructure was critical for diagnosing message persistence issues and validating the effectiveness of the three-stage response flow implementation that resolved user experience problems.

### Root Cause Investigation Framework
When performance issues arise, I apply systematic hypothesis-driven analysis:

**Hypothesis 1: Model Generation Bottleneck**
- Analysis of generation parameters (max_new_tokens, temperature, top_p)
- Identification of suboptimal settings causing delays

**Hypothesis 2: LangGraph Workflow Overhead**
- State serialization/deserialization analysis
- Memory checkpointing latency assessment
- Tool node processing delay measurement

**Hypothesis 3: Resource Contention**
- GPU memory fragmentation monitoring
- CUDA context switching analysis
- Quantization overhead assessment during generation

---

## Automated Evaluation Capability (OpenAI Evals API)

While I have not implemented automated evaluation frameworks for Mimir, I maintain the capability to deploy systematic prompt evaluation using APIs like OpenAI Evals. This approach would be valuable for large-scale testing scenarios or when developing prompts that require extensive validation across diverse edge cases.

### Task Description
I begin by defining the task for the model to complete. In the example below, I employ a persona pattern combined with a classification pattern to create clear, structured instructions.

```python
from openai import OpenAI
client = OpenAI()

instructions = """
You are a customer service professional skilled in identifying client needs. Given a user message, categorize it using one of these options: "Delivery Issue(s)," "Product Issue(s)," "Subscription Modify/Cancel," "Account Issue(s)," or "Undefined/General." Respond with only one category that best matches the user message. Use "Undefined/General" only when no other options apply.
"""

user_message = "I got an email that my account was locked from too many login attempts."

response = client.responses.create(
    model="gpt-4.1",
    input=[
        {"role": "developer", "content": instructions},
        {"role": "user", "content": user_message},
    ],
)

print(response.output_text)
```

The model should identify that this user message belongs to the `Account Issue(s)` category, as the input indicates either an unauthorized login attempt on the user's account or the user accidentally locking themselves out.

### Evaluation Setup
With the task defined, I can now set up an evaluation via the API. This requires defining a schema for test data and establishing graders to determine whether the model output is correct.

```python
from openai import OpenAI
client = OpenAI()

eval_obj = client.evals.create(
    name="Customer Query Categorization",
    data_source_config={
        "type": "custom",
        "item_schema": {
            "type": "object",
            "properties": {
                "user_message": { 
                    "type": "string",
                    "description": "The customer's message or query"
                },
                "correct_label": { 
                    "type": "string",
                    "description": "The expected category classification",
                    "enum": [
                        "Delivery Issue(s)",
                        "Product Issue(s)", 
                        "Subscription Modify/Cancel",
                        "Account Issue(s)",
                        "Undefined/General"
                    ]
                }
            },
            "required": ["user_message", "correct_label"]
        },
        "include_sample_schema": True
    },
    testing_criteria=[
        {
            "type": "string_check",
            "name": "Match output to human label",
            "input": "{{ sample.output_text }}",
            "operation": "eq",
            "reference": "{{ item.correct_label }}"
        }
    ]
)

print(f"Created eval with ID: {eval_obj.id}")
```

### Test Cases
Test cases, with human labels, are used to evaluate the model's performance. In a real-world scenario, I'd have many more to better gauge the model's performance. In this example, I would be particularly interested in targeting edge cases where the model may apply the `Undefined/General` label when a more descriptive category is applicable, or not apply it when expected.

One such edge case is shown below, where the user query is "Update email address," a message a user may realistically send. For this specific case, I am targeting a foreseen issue where I anticipate the model may apply the `Account Issue(s)` category despite the query not indicating an issue, but rather a general guidance request on how to update the email.

Create a JSONL file named `usermessages.jsonl`:

```jsonl
{"item": {"user_message": "I got an email that my account was locked from too many login attempts.", "correct_label": "Account Issue(s)"}}
{"item": {"user_message": "My order hasn't arrived yet and it's been 2 weeks", "correct_label": "Delivery Issue(s)"}}
{"item": {"user_message": "The product I received is damaged", "correct_label": "Product Issue(s)"}}
{"item": {"user_message": "I want to cancel my subscription, this costs too much.", "correct_label": "Subscription Modify/Cancel"}}
{"item": {"user_message": "Customer service number please", "correct_label": "Undefined/General"}}
{"item": {"user_message": "Update email address", "correct_label": "Undefined/General"}}
```

#### Upload Test Data via API

```bash
curl https://api.openai.com/v1/files \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -F purpose="evals" \
  -F file="@usermessages.jsonl"
```

```python
file = client.files.create(
    file=open("usermessages.jsonl", "rb"),
    purpose="evals"
)

print(f"Uploaded file with ID: {file.id}")
```

#### Create and Run the Evaluation

Taking note of the unique ID produced by the file upload response, create an eval run:

```bash
curl https://api.openai.com/v1/evals/YOUR_EVAL_ID/runs \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{
        "name": "Categorization test run",
        "data_source": {
            "type": "responses",
            "model": "gpt-4.1",
            "input_messages": {
                "type": "template",
                "template": [
                    {"role": "developer", "content": "You are a customer service professional skilled in identifying client needs. Given a user message, categorize it using one of these options: \"Delivery Issue(s),\" \"Product Issue(s),\" \"Subscription Modify/Cancel,\" \"Account Issue(s),\" or \"Undefined/General.\" Respond with only one category that best matches the user message. Use \"Undefined/General\" only when no other options apply."},
                    {"role": "user", "content": "{{ item.user_message }}"}
                ]
            },
            "source": { "type": "file_id", "id": "YOUR_FILE_ID" }
        }
    }'
```

```python
run = client.evals.runs.create(
    eval_id="YOUR_EVAL_ID",
    name="Categorization test run",
    data_source={
        "type": "responses",
        "model": "gpt-4.1",
        "input_messages": {
            "type": "template",
            "template": [
                {"role": "developer", "content": "You are a customer service professional skilled in identifying client needs. Given a user message, categorize it using one of these options: \"Delivery Issue(s),\" \"Product Issue(s),\" \"Subscription Modify/Cancel,\" \"Account Issue(s),\" or \"Undefined/General.\" Respond with only one category that best matches the user message. Use \"Undefined/General\" only when no other options apply."},
                {"role": "user", "content": "{{ item.user_message }}"},
            ],
        },
        "source": {"type": "file_id", "id": "YOUR_FILE_ID"},
    },
)

print(f"Created run with ID: {run.id}")
```

### Analyzing Results

To check the status, a script like the one below can be run:

```python
run_status = client.evals.runs.retrieve("YOUR_EVAL_ID", "YOUR_RUN_ID")
print(f"Status: {run_status.status}")
print(f"Results: {run_status.result_counts}")
```

The completed run will provide total test cases processed, pass/fail counts for each testing criterion, token usage statistics, and detailed results available in the dashboard via a `report_url`. Using this, I can assess my prompt to see where the model struggles, and perhaps integrate a new pattern or examples to aid the model in correctly labeling user input in a customer service scenario such as this one. 

### Applying Assessment

Prompt engineering is iterative. For demonstration, let's say the aforementioned foreseen issue came to fruition when running the test cases. In which case, I may add another layer to the system prompt to better handle edge cases, such as examples, add a clear definition of when to use `Undefined/General`, integrate another prompting pattern, or some combination.

#### Example Revised Prompts

##### Version One (Original)
You are a customer service professional skilled in identifying client needs. Given a user message, categorize it using one of these options: "Delivery Issue(s)," "Product Issue(s)," "Subscription Modify/Cancel," "Account Issue(s)," or "Undefined/General." Respond with only one category that best matches the user message. Use "Undefined/General" only when no other options apply.

##### Version Two
You are a customer service professional skilled in identifying client needs. Given a user message, categorize it using one of these options: "Delivery Issue(s)," "Product Issue(s)," "Subscription Modify/Cancel," "Account Issue(s)," or "Undefined/General." Use "Undefined/General" only when about: account updates, general questions, messages not indicative of a customer issue, or otherwise unrelated to the other categories. Respond with only one category that best matches the user message.

##### Version Three
You are a customer service professional skilled in identifying client needs. Given a user message, follow this categorization process:

Step 1: Initial Classification
First, categorize the user message as one of the following:
- "Customer Issue"
- "Subscription Change"
- "Other"

Step 2: Final Classification
- If the user query is in the "Customer Issue" category, assign one of: "Delivery Issue(s)," "Product Issue(s)," or "Account Issue(s)"
- If the user message is related to a subscription change (rather than an issue with billing or other general update), assign: "Subscription Modify/Cancel"
- For anything else initially considered as "Other," assign: "Undefined/General"

Task: Categorize the given user message and respond with only the final category that best matches.
