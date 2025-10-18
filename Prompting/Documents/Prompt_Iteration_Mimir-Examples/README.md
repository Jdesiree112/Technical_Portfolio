# Prompt Iteration
## Overview
Critical to any AI project, prompt iteration is the process of refining prompts through a series of back-and-forth messages with an AI, with interwoven adjustments for improvement. This collection follows the process I developed to iterate on the system prompts, prompt parts used in Mimir, f-string prompt templating, and Jinja message construction. This is intended to be a live document, tracking iterations over time as Mimir is improved.

## Prompt Testing Interface
To support systematic prompt iteration and evaluation, Mimir includes a dedicated Prompt Testing Interface accessible via the application's multi-page Gradio interface. This tool enables isolated testing of individual agent configurations without requiring full application deployment or manual code modification.

### Purpose
The Prompt Testing Interface serves as an integrated development environment for prompt engineering, providing:
- Rapid iteration on prompt segments without application restart
- Quantitative evaluation of prompt performance across batch inputs
- Comparative analysis of different model configurations
- Reproducible testing through JSON export of results and configurations

### Architecture
The interface operates independently of the main chatbot pipeline, allowing direct access to:
- **Tool Decision Agent** (Shared Mistral-7B)
- **Routing Agents 1-4** (Shared Mistral-7B)
- **Thinking Agents** (Shared Mistral-7B for QA/Reasoning, GGUF for Math)
- **Response Agent** (Fine-tuned Phi-3)

Each agent class exposes its available prompt segments as configurable checkboxes, with default model parameters automatically populated based on production configurations.

### Key Features

#### Configuration Panel
- **Model Selection**: Dropdown menu for selecting agent class to test
- **Prompt Segments**: Dynamic checkboxes displaying available prompt parts for selected model
- **Model Parameters**: Adjustable temperature (0.0-2.0), top-p (0.0-1.0), and top-k (1-100) sliders with defaults matching production settings

#### Batch Processing
- **Prompt Entry Table**: Up to 50 prompts can be entered for batch processing
- **Passing Criteria**: Three evaluation modes per prompt:
  - `disabled`: No evaluation criteria
  - `includes text`: Response must contain specified text (case-insensitive)
  - `exact text`: Response must exactly match specified text (case-insensitive)
- **Batch Execution**: Sequential processing with real-time progress tracking

#### Analytics Dashboard
Provides tabular analysis of batch results including:
- **Per-Prompt Metrics**: Response time (seconds), input token count, pass/fail status
- **Aggregate Statistics**: Average response time, average token count, accuracy rate (percentage of prompts meeting passing criteria)
- **Result Export**: Complete test session exportable as timestamped JSON file containing:
  - Configuration parameters (model, temperature, top-p, top-k)
  - Full prompt and response pairs
  - Individual and aggregate metrics
  - ISO 8601 timestamps for reproducibility

### Workflow Integration
The Prompt Testing Interface fits into the prompt iteration workflow as follows:

1. **Hypothesis Formation**: Identify prompt segment requiring improvement based on production analytics or user feedback
2. **Isolated Testing**: Load relevant agent class in testing interface and configure prompt segments
3. **Batch Evaluation**: Process representative set of user queries with passing criteria based on expected behavior
4. **Metric Analysis**: Review response times, accuracy rates, and qualitative outputs
5. **Iteration**: Adjust prompt segments, model parameters, or prompt combinations and repeat
6. **Documentation**: Export successful configurations as JSON for version control and comparison
7. **Deployment**: Update `prompt_library.py` with validated prompt improvements

This systematic approach replaces ad-hoc prompt modification in code, reducing iteration cycle time from minutes (rebuild/redeploy) to seconds (configuration change in interface).

### Library Versions
1. [v2.0 Prompt Library](https://github.com/Jdesiree112/Technical_Portfolio/blob/main/Prompting/Documents/Prompt_Iteration_Mimir-Examples/Prompt_Library_v2.0/Mimir_Prompts.md)
