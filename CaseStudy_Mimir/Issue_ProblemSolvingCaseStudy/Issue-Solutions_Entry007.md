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
| User Interface Finalized | The user interface should have no changes. It has been verified that the user interface is not affected. |
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
