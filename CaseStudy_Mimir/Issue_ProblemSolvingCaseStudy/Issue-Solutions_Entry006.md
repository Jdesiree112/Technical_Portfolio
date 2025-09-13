## Faulty Decision Logic

After a manual end-to-end review, I identified inefficiencies and gaps in the existing decision logic that could lead to inaccurate system prompt snippet allocation. The current implementation relies heavily on regex-based rules to qualify prompts, which are brittle and difficult to maintain as the logic grows more complex.

To address this, I plan to adopt a **machine learning approach** by implementing a **decision tree classifier** using scikit-learn. Rather than relying on static regex patterns, the classifier will learn decision boundaries from a curated corpus of prompt examples paired with their correct system prompt outputs. This will allow the model to automatically classify incoming prompts and predict the ideal system prompt snippet for producing the most contextually accurate responses.

**Expected Benefits:**

* **Improved accuracy:** The tree will learn from labeled examples, reducing the risk of misclassification caused by edge cases not captured by regex.
* **Interpretability:** Decision trees produce human-readable rules that can be inspected to verify that the model mirrors the intended logic.
* **Maintainability:** Updating the logic becomes as simple as adding or adjusting training examples and retraining, rather than rewriting regex rules.
* **Scalability:** The approach can easily accommodate new prompt types and system actions as the corpus grows.

---

### Examples, Problems, Solutions, and Successes

```
def _should_use_conversational_tone(user_input: str, conversation_state: list) -> bool:
    """Helper to determine if conversational tone is more appropriate than academic"""
    
    # Simple responses to gratitude, confirmations, etc.
    conversational_responses = [
        r'(thank\s+you|thanks)',
        r'(that\s+helps?|that\s+makes\s+sense)',
        r'(got\s+it|understand|i\s+see)',
        r'(perfect|great|awesome|cool)',
        r'(yes\s*,?\s*please|no\s*,?\s*thanks?)'
    ]
```
Note that the helper tool for the conversational style prompt checks for common phrases that may be mixed into user prompts that are not conversational. A remediation I considered was to add layers to this function to filter for short responses where there is no recent history indicating the user prompt is a follow-up.

```
def _needs_discovery_phase(user_input_lower: str, conversation_state: list, is_conversation_start: bool, 
                          is_greeting: bool, is_casual_conversation: bool) -> bool:
    """
    Determine if the user needs the discovery phase to understand their learning goals.
    Discovery is triggered when:
    1. Conversation starts with educational intent but unclear goals
    2. User mentions wanting to lear,n but is vague about specifics
    3. User asks for help but hasn't specified their knowledge level
    """
```
The function correctly disqualifies simple greeting or conversational user prompts. However, the logic assumes the discovery prompt would only be useful in early conversation, which is not true. The user may explore a topic with the model, then in the later stages of the conversation propose a new topic or pivot. In cases such as this, the chatbot should identify that a new topic is being discussed and strive to understand what the user knows before blindly teaching the user about the topic. 

Reviewing just the regex patterns, while efficient, there are gaps in many areas that could lead to incorrect prompt use or prompts that should be used not being used. The concept of having a variety of small system prompts is to reduce the instructional burden on the model, adding only relevant guidelines to the user prompt each turn. If unnecessary prompts are added, or needed prompts are not added, then responses will not meet the baseline standard of quality and hinder user experience. These issues prompted me to explore new approaches to defining when prompts should be applied to control the conversation flow effectively and improve user experience. Buddled into this, I also set out to integrate the tool decision engine into the prompt selection process rather than leaving it separate. To begin, I'll begin with defining the **user story** and **Definition of Done (DoD)** for this objective.

### User Story

**As a student, I want to receive a situationally tailored, personalized conversation to aid in my learning process.**

### DoD

| Activity | Description |
| :-: | :- |
| Code Complete | All coding tasks are complete, with needed changes implemented. The LangChain and LangGraph portions of the application have been updated to function with the revised code. All code is functionally complete. |
| Unit Testing Passed | Test user prompts yield the expected resulting parts consistently and with 98% accuracy. |
| Functional Requirements Met | The code results in an effective conversational flow, applying reasonable and accurate prompt parts to user prompts. |
| Design and Architecture | The design and architecture are compatible with the deployment environment, Hugging Face spaces, and are functional with existing LangChain and LangGraph orchestration. |
| Code Review | Code has been reviewed and meets the standard for this project.|
| Documentation Updated | This Issue and Solution report has been updated to include results and reflections. API Documentation has been revised. |
| Security Measures and Considerations | The new system has been checked for vulnerabilities |
| User Interface Finalized | The user interface should have no changes. It has been verified that the user interface is not affected.
| Regression Testing | Mimir has been tested to ensure no regression of the application function. |
| Deployment Ready | The code is ready to deploy in HuggingFace Spaces. |
| Technical Debt Evaluation | The code and automated evaluations and/or metrics tracking have been assessed, ensuring no unnecessary technical debt. All needed revisions have been made. |

### Revised Decision Tree Outline
This decision tree was constructed in an effort to visualize, evaluate, and expand on the decision process. This was somewhat exploratory, involving asking "What would a student or teacher expect from the model?" 

<img width="1859" height="2336" alt="Decision Flow Chart" src="https://github.com/user-attachments/assets/7fa0da78-e943-4866-98aa-e755ba7868fa" />
**Made with Miro** https://miro.com/app/board/uXjVJIvsP1I=/?share_link_id=557918991169

## Machine Learning Implementation Architecture

The new decision logic will **mostly** replace the current regex-based approach with a **scikit-learn Decision Tree Classifier** trained on the `Mimir_DecisionClassifier` dataset hosted on Hugging Face. This system takes note of the success achieved by the model automated `Tool_Decision_Engine` but extends it to handle all prompt segment decisions in a unified, intelligent manner. Rather than running a query, this system will train a decision tree classifier with user messages and calculated values such as conversation length and `Tool_Decision_Engine`. This system blends redex filtering, conversation state tracking, and ML principles together for a more fluid prompt assignment system.

### System Architecture Overview

**Core Components:**
```
app.py                     # Main application with LangChain/LangGraph orchestration
prompt_classifier.py      # ML-based decision engine (new)
Mimir_DecisionClassifier   # Hugging Face dataset with labeled training examples
```

**Decision Flow Integration:**
The new system will replace the `determine_prompt_segments()` function with a single ML model call that predicts all required prompt segments simultaneously, eliminating the need for separate context analysis functions with minimal complex logic functions for some input variables.

### Technical Implementation Details

**Dataset Structure (`Mimir_DecisionClassifier`):**
```python
# Expected dataset schema
{
    "user_input": "Can you help me with calculus?",
    "conversation_length": 1,
    "is_first_turn": True,
    "input_character_count": 31,
    "contains_greeting": False,
    "contains_educational_keywords": True,
    "topic_change_detected": False,
    "recent_discovery_questions": 0,
    "prompt_segments": ["CORE_IDENTITY", "DISCOVERY_MODE"]  # Multi-label target
}
```
**Note, this data is in a JSONL format in the database, but is presented in JSON here for readability.**

**Feature Engineering Pipeline:**
The `prompt_classifier.py` is held in a separate file in the application root for ease of editing. As Mimir has grown, so has the `app.py`, making this a necessary choice as opposed to adding extra verbosity to the main code file. 

Input data is outlined below, with notes explaining the source and method of acquisition for each value. All input values derive from the current conversation state, with one being the output of the `tool_decision_engine`, three being redex logic based on the current prompt, and all others being variables tracked within the `app.py` per conversation.

```python
@dataclass
class ConversationInput:
    """Input data structure for the classifier - all values pre-calculated by app.py"""
    user_input: str
    conversation_length: int                    # Total user prompts sent (tracked in app.py)
    is_first_turn: bool                        # Tracked in app.py (starts True, becomes False after first response)
    input_character_count: int                 # Length of user prompt
    is_short_input: bool                       # True if user prompt ≤6 chars, False if >6 chars
    recent_discovery_count: int                # Count tracked in app.py
    contains_greeting: bool                    # From regex logic
    contains_educational_keywords: bool        # From regex logic
    requires_visualization: bool               # Yes/No from tool decision engine
    topic_change_detected: bool                # From regex logic
```

The output of the classification system is the prompts, indicated as true/false using bool values, the confidence score for evaluation, and the decision time for efficiency tracking. Additional evaluation measures are in place to assess the accuracy of the model as it is currently, as well as how it improves with additional data being added. The training data is split 70% for training and 30% for testing in a random state of 42. Loggers are implemented throughout the full pipeline to track accuracy.

**Multi-Label Classification Approach:**
Unlike the current either/or logic, the ML model will use **Multi-Label Classification** to predict combinations of prompt segments. This allows for nuanced decisions like:
- `["CORE_IDENTITY", "CONVERSATIONAL"]` for simple greetings
- `["CORE_IDENTITY", "DISCOVERY_MODE"]` for vague educational requests
- `["CORE_IDENTITY", "GUIDING_TEACHING", "TOOL_USE_ENHANCEMENT"]` for complex topics requiring visualization

**Integration with Hugging Face Dataset:**
```python
from datasets import load_dataset
from sklearn.tree import DecisionTreeClassifier
from sklearn.multioutput import MultiOutputClassifier

class PromptClassifier:
    def __init__(self):
        # Load training data from Hugging Face
        self.dataset = load_dataset("jdesiree/Mimir_DecisionClassifier")
        self.feature_extractor = ConversationFeatureExtractor()
        self.model = self._train_classifier()
        
    def _train_classifier(self):
        # Convert dataset to feature matrix and multi-label targets
        X, y = self._prepare_training_data()
        
        # Use MultiOutputClassifier for multi-label prediction
        classifier = MultiOutputClassifier(
            DecisionTreeClassifier(
                criterion='entropy',
                max_depth=8,
                min_samples_split=5,
                random_state=42
            )
        )
        return classifier.fit(X, y)
```

**Unified Decision Interface:**
The new system will provide a clean interface that replaces your current `determine_prompt_segments()` function:

```python
def determine_prompt_segments(user_input: str, conversation_state: list) -> str:
    """
    ML-powered prompt segment selection replacing regex-based logic.
    """
    start_time = time.perf_counter()
    
    # Extract features and predict segments
    features = classifier.extract_features(user_input, conversation_state)
    predicted_segments = classifier.predict_segments(features)
    
    # Log decision for evaluation
    end_time = time.perf_counter()
    decision_time = end_time - start_time
    
    log_metric(f"ML decision: {predicted_segments} | Decision time: {decision_time:0.4f}s | "
              f"Input: '{user_input[:30]}...'")
    
    # Combine selected segments
    return classifier.combine_segments(predicted_segments)
```

### Advantages Over Current System

**Improved Accuracy and Flexibility**: Learning from labeled examples eliminates edge cases that regex patterns miss, such as:
- Mixed casual/educational inputs: "Hey, can you help me understand derivatives?"
- Topic changes mid-conversation: "Actually, let's switch to chemistry instead."
- Nuanced discovery needs: User provides partial context but needs clarification

**Unified Tool Integration**: The model will directly predict when `TOOL_USE_ENHANCEMENT` is needed, leveraging the output of the already successful `Tool_Decision_Engine` as input. This is expected to also influence the decision process for teaching or guiding tasks, as the necessity for tools is linked to instructional tasks requiring visuals or graphics needed for practice questions. It is important to note that not all `GUIDING_TEACHING` tasks will need `TOOL_USE_ENHANCEMENT`, but all `TOOL_USE_ENHANCEMENT` tasks will need `GUIDING_TEACHING`.

**Maintainable Logic**: Updates require adding training examples to the Hugging Face dataset and retraining, rather than debugging complex regex patterns and conditional statements. The inclusion of conversational language detection in a system such as this, rather than as a standalone regex, serves to add to the model's understanding of when a prompt should apply, rather than disqualifying a needed prompt in a rigid system.

**Interpretable Decisions**: Decision trees provide a clear visualization of learned rules, allowing human evaluation of the model's logic. This maintains ease of manual evaluation and facilitates future adjustments as continued quality assurance testing uncovers currently unseen issues or areas of improvement.

**Performance Monitoring**: Built-in logging and metrics collection enable continuous improvement through:
- Classification accuracy tracking
- User satisfaction correlation analysis
- A/B testing against rule-based baselines
- Automated retraining pipelines

### Expected Implementation Timeline

**Phase 1**: Create `prompt_classifier.py` with basic feature extraction and model training using the `Mimir_DecisionClassifier` dataset

**Phase 2**: Replace `determine_prompt_segments()` in `app.py` with ML-based predictions while maintaining backward compatibility

**Phase 3**: Remove deprecated regex-based functions and integrate performance monitoring

**Phase 4**: Implement a continuous learning pipeline for model updates based on production data. User data will not be used, but rather data curated through testing sessions and evaluated, with failed areas corrected, then added to the training data to train out similar flaws.

This ML-driven approach directly addresses the gaps identified in the existing system while maintaining what originally worked well. This is not meant to be a complete overhaul of all logic, but rather an enhancement of the current system.
