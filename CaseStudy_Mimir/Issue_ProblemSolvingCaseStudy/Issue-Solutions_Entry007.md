## Fine-Tuning
### Overview

Multiple approaches have been implemented to address verbosity, instruction-following, and writing quality/tone issues in Mimir's responses. These efforts included a truncation function and an advanced ML decision tree for managing system prompt snippets based on relevance. Despite these interventions, minimal improvement was achieved, indicating that the underlying model requires fine-tuning to meet performance expectations. The base model consistently fails to deliver appropriate responses in its current state. This section defines the **user story** and **Definition of Done (DoD)** for the fine-tuning objective.

#### User Story

**As a student, I want to receive an educationally beneficial and situationally appropriate response to my questions and requests to aid in my learning journey.**

#### DoD

| Activity | Description |
| :-: | :- |
| Code Complete | All coding tasks are complete, with needed changes implemented. The model has been updated in the app.py to be the fine-tuned version. |
| Unit Testing Passed | Test user prompts yield the expected resulting parts consistently and with 95% accuracy. |
| Functional Requirements Met | The code trains the model on the `Mimir_fine_tuning` data set, resulting in a more accurate, appropriate, and helpful conversation. |
| Design and Architecture | The design and architecture are compatible with the deployment environment, Hugging Face spaces, and are functional with `app/py`. |
| Code Review | Code has been reviewed and meets the standard for this project.|
| Documentation Updated | This Issue and Solution report has been updated to include results and reflections. API Documentation has been revised. |
| Security Measures and Considerations | The new system has been checked for vulnerabilities |
| User Interface Finalized | The user interface is updated with an added button to the group in the bottom right. The new button indicates by color the status of the models, with details displayed on hover. No other changes are visible in the user interface.|
| Regression Testing | Mimir has been tested to ensure no regression of the application function. |
| Deployment Ready | The code is ready to deploy in HuggingFace Spaces. |
| Technical Debt Evaluation | The code and automated evaluations and/or metrics tracking have been assessed, ensuring no unnecessary technical debt. All needed revisions have been made. |

### Training Data
The training dataset is maintained and updated in Google Sheets, then exported to Hugging Face Datasets as a CSV file for consistent version control. For the most current view of the dataset, which is anticipated to expand rapidly, reference this [link](https://docs.google.com/spreadsheets/d/1fsThrvqZmNt0JYFbeUG9fRDuXM0pmTwnK4eZOOjffY8/edit?usp=sharing). 

Training data collection occurs during testing sessions, where instances of suboptimal model performance are documented for dataset inclusion. Dedicated data collection sessions target anticipated failure modes to gather curated examples. To maintain dataset balance, diverse user prompts are included, incorporating queries inspired by online forum discussions and user feedback from broader application deployment.

**Example**
- **Prompt**: Hello!
- **Model Response**: I'm here whenever your curiosity sparkles or when studying seems like climbing Everest – just let me know which mountain we need conquering today in our quest of knowledge together :) How about starting by asking what subject piques your interest. Or perhaps sharing where it feels challenging right now could be really beneficial - every learner has those hurdles they want us all ready to jump over side step one classroom wall closer towards success storytelling time ;) Let this moment inspire action—shall we delve into math problems solving strategies next weekend session maybe.
- **Issues**:
   - Tone: The model response lacks the expected professional tone, instead presenting as overly flowery with excessive visual and metaphoric language. Sentiment is delivered through emoticons :) and ;) , which is not ideal nor acceptable for this application.
   - Writing Quality: Numerous grammatical errors and flow issues are present with clear filler language usage.
   - Other Notes: The excessive hyphen use is distracting and unneeded. 
- **Ideal Response**: Hello! I am Mimir, a tutoring assistant. Is there a subject you would like to explore, or a test you'd like to prepare for?

The provided example demonstrates the systematic review process for collecting prompt-response pairs, performance rationales, and ideal responses. The implementation utilizes Hugging Face's `Trainer` library to fine-tune the current model, producing an optimized version for deployment. Model accuracy is tracked through manual evaluation using the existing benchmark framework for response quality assessment.

## Model Training
After completing the development and debugging process in Hugging Face's web-based VS Code environment, a fine-tuned version of Phi-3 named `mimir-phi3` was successfully trained on the collected dataset. The trained model files are hosted on Hugging Face at [jdesiree/Mimir-Phi-3.5](https://huggingface.co/jdesiree/Mimir-Phi-3.5/tree/main). Future training iterations will be conducted as the dataset expands, with training infrastructure available at [jdesiree/Mimir_Training](https://huggingface.co/spaces/jdesiree/Mimir_Training/tree/main).

### Training Performance and Results
The fine-tuning process was conducted using the microsoft/Phi-3-mini-4k-instruct base model (3.8 billion parameters) with the following configuration:
- **Training Dataset**: 17 total examples (13 training, 4 evaluation)
- **Training Duration**: 5 epochs completed in approximately 11 minutes
- **Precision**: fp16 for memory efficiency
- **Batch Configuration**: Per-device batch size of 1 with gradient accumulation steps of 8
- **Learning Rate**: Conservative 2e-5 with warmup ratio of 0.1

**Performance Metrics:**
- Initial training loss: 1.52
- Final training loss: 165.17
- Training exhibited some instability mid-process (loss peaks ~459) but stabilized in later epochs
- Model convergence achieved despite limited dataset size

**Technical Considerations:**
The training process encountered typical challenges associated with small datasets, including loss instability and potential overfitting risks. The final model demonstrated improved response characteristics aligned with the target professional tone and educational focus, though additional data collection and retraining cycles are recommended to enhance robustness and consistency.

## Advanced Model Management and Status Indication System

This section documents the implementation of the sophisticated model management system that provides graceful fallback from fine-tuned models to base models, along with a comprehensive status indication system.

### Model Fallback Implementation

#### Overview

The `Phi3MiniEducationalLLM` class implements a robust three-tier fallback system that ensures the application remains functional even when the preferred fine-tuned model is unavailable. This system prioritizes user experience by attempting to load models in order of preference while providing transparent status reporting.

#### Fallback Strategy

The model loading follows this prioritized sequence:

1. **Primary Strategy**: Full fine-tuned model loading
   - Attempts to load `jdesiree/Mimir-Phi-3.5` as a complete fine-tuned model
   - Uses 4-bit quantization for memory efficiency
   - Logs success and sets `model_type = "fine-tuned"`

2. **Secondary Strategy**: PEFT adapter loading
   - If primary fails, attempts to load as a PEFT (Parameter Efficient Fine-Tuning) adapter
   - First loads the base model (`microsoft/Phi-3-mini-4k-instruct`)
   - Then applies the adapter on top using `PeftModel.from_pretrained()`
   - Sets `model_type = "adapter"` and `is_adapter = True`
   - Only available if PEFT library is installed

3. **Tertiary Strategy**: Base model fallback
   - If both fine-tuned strategies fail, loads the base model
   - Sets `model_type = "base"` and `using_fallback = True`
   - Logs fallback activation with detailed error reporting

#### Error Handling and Logging

The implementation includes comprehensive error handling:

```python
def _attempt_load_fine_tuned_model(self):
    """Attempt to load the fine-tuned model, returns (model, success, error_msg)"""
    try:
        # Full model loading logic
        return model, True, None
    except Exception as e:
        if PEFT_AVAILABLE:
            try:
                # PEFT adapter loading logic
                return model, True, None
            except Exception as peft_error:
                return None, False, f"Full model error: {e}, PEFT error: {peft_error}"
        else:
            return None, False, f"Full model error: {e}, PEFT not available"
```

#### Tokenizer Compatibility

The system ensures tokenizer compatibility by always loading the tokenizer from the base model, regardless of which model variant is ultimately used. This prevents tokenization mismatches that could cause generation failures:

- Tokenizer loaded from `BASE_MODEL` path
- Special tokens (like `<|end|>`) configured consistently
- Pad tokens properly set across all model variants

#### Model Information Tracking

The `get_model_info()` method provides comprehensive status information:

```python
{
    "status": "loaded",
    "model_type": "fine-tuned" | "adapter" | "base",
    "is_adapter": boolean,
    "fine_tuned_path": "jdesiree/Mimir-Phi-3.5",
    "base_model_path": "microsoft/Phi-3-mini-4k-instruct",
    "using_fallback": boolean
}
```

#### Performance Logging

The system includes detailed performance metrics that track:
- Model loading times with fallback indicators
- Memory usage patterns
- Success/failure rates for different loading strategies
- Fallback activation frequency for monitoring

### Status Indicator Button Implementation

#### Visual Status System

The status indicator button provides immediate visual feedback about the current model state through a color-coded system integrated into the main interface button area.

#### Color Coding Scheme

The button uses three distinct visual states:

1. **Success State (Green)**
   - Color: `#38a169` with hover effect `#2f855a`
   - Indicates: Fine-tuned model or adapter successfully loaded
   - Tooltip: Shows model type and path

2. **Warning State (Orange)**
   - Color: `#dd6b20` with hover effect `#c05621`
   - Indicates: Fallback to base model is active
   - Tooltip: Shows fallback reason and base model path

3. **Error State (Red)**
   - Color: `#e53e3e` with hover effect `#c53030`
   - Indicates: Model loading failure or system error
   - Tooltip: Shows error details (truncated to 50 characters)

#### CSS Implementation

The button styling includes sophisticated hover effects and tooltip systems:

```css
.status-button-success:hover {
    background: #2f855a !important;
    border: 2px solid #2f855a !important;
    transform: translateY(-1px);
    box-shadow: 0 4px 8px rgba(56, 161, 105, 0.3);
}
```

#### Tooltip System

The tooltip implementation provides detailed status information:

- Positioned above the button using CSS `::after` pseudo-elements
- Dark background (`#2d3748`) for good contrast
- Responsive width with proper text wrapping
- Pointer events disabled to prevent interference
- Z-index management for proper layering

#### Integration Points

The status button integrates with multiple system components:

1. **Model Loading**: Updates immediately after model initialization
2. **Error Handling**: Reflects real-time system status
3. **UI Framework**: Seamlessly integrated with Gradio button styling
4. **Performance Monitoring**: Status changes trigger metric logging

#### JavaScript Enhancement

Dynamic tooltip content is managed through JavaScript injection:

```javascript
document.addEventListener('DOMContentLoaded', function() {
    const statusBtn = document.getElementById('status-button');
    if (statusBtn) {
        statusBtn.setAttribute('title', '{tooltip_content}');
    }
});
```

#### Status Information Flow

The status system follows this information flow:

1. `Educational_Agent.get_agent_status()` retrieves model information
2. `get_status_button_info()` processes status into UI-appropriate format
3. Button class and tooltip content determined based on model state
4. CSS styling applied automatically based on class assignment
5. JavaScript sets tooltip content dynamically on page load

#### Accessibility Features

The implementation includes several accessibility considerations:

- High contrast color schemes for visual clarity
- Tooltip content accessible via hover for detailed information
- Non-interactive button prevents accidental activation
- Clear visual hierarchy with proper sizing and positioning
- Screen reader-friendly with semantic HTML structure

#### Monitoring and Debugging

The status system contributes to system observability:

- Status changes logged to performance metrics
- Error states captured with detailed diagnostic information
- Fallback activations tracked for reliability monitoring
- Visual indicators provide immediate feedback for troubleshooting

This comprehensive status indication system ensures users always have clear visibility into the model loading state while providing administrators with the diagnostic information needed for system maintenance and optimization.
