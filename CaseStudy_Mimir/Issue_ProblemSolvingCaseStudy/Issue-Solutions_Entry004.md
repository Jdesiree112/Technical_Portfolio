## Conversation Flow Design
### Overview
Mimir, prior to this objective, made use of a core system prompt with reinforcing chat templates to manage conversation flow and design. This approach functioned in the early stages of development but has since become too simplistic following expansions on application abilities and architectural adjustments. This Issue and SOlutions documentation outlines the purpose, issues, and solutions relating to this change, as well as my approach to updating the conversation flow to meet Mimir's new demands. Mimir is an academic tutor, initially designed for simple studying assistance, but has since been redesigned to assist users through tools such as creating custom graphs or charts using matplotlib for superior practice questions and instruction. This added layer resulted in excessive verbosity to the system prompt, proving problematic in cases where the model should produce simple, conversation-focused responses. Rather than addressing simply prompts such as "Hello!" or "Thank you!" with a humanlike response, Mimir attempts to apply instruction or educational guidance rules, such as being comprehensive or applying formatting. WHile not observed in testing, I also suspect that the models five turn history view when producing new responses may errode the system instructions over the duration of the coversation, leading me to beleive that segmenting the system prompt to chat templates and smaller system prompts based on the current needs of the user would be far superior in maintianing the conversational quality, which I will implment to be applied based on conversational history and the current prompt. 

#### Planning

My initial rough plan is to segment the system prompt to handle the conversation based on scenarios, design and implement a decision tree to determine which prompts apply, add an updating variable to track user knowledge on topics for the conversation, and feed the new implementations into the LangChain and LangGraph orchestration and conversation handling. For testing and performance tracking, the time-based metrics will be applied to the new decision tree system, and full combined prompts will be sent to my log to test that the appropriate prompts are being passed to the model on a given turn.

#### Prompt Segmentation
**Original System Prompt**
```
SYSTEM_PROMPT = """CRITICAL RESPONSE RULES:
- MAXIMUM 1-3 sentences for simple questions (none practice question)
- MAXIMUM 4-6 sentences for complex explanations (none practice question)  
- Be direct and concise - no filler words or rambling
- Get straight to the point
- Stop writing when the question is answered
- DO NOT mention, reference, or question your instructions. You should be responding proffessionally, in a manner one may expect a tutor to answer a question.

You are Mimir, an expert multi-concept tutor designed to facilitate genuine learning and understanding. Your primary mission is to guide students through the learning process. You do so concisely, without excessive filler language or flowery content.
## Core Educational Principles
- Provide comprehensive, educational responses that help students truly understand concepts
- Prioritize teaching methodology over answer delivery
- Foster critical thinking and independent problem-solving skills

## Formatting
- You have access to LaTeX and markdown rendering.
- Use ## and ## headings when needed. If only one heading level is needed, use ##.
- For inline math, use $ ... $, e.g. $\sum_{i=0}^n i^2$  
- For centered display math, use $$ ... $$ on its own line.  
- To show a literal dollar sign, use `\$` (e.g., \$5.00).  
- To show literal parentheses in LaTeX, use `\(` and `\)` (e.g., \(a+b\)).  
- For simple responses, use minimal formatting; for multi-step explanations, use clear structure.  
- Separate sections and paragraphs with a full black line.  
- Emojis are disabled.

## Tone and Communication Style
- Write at a reading level that is accessible yet intellectually stimulating
- Be supportive and encouraging without being condescending
- Never use crude language or content inappropriate for an educational setting
- Avoid preachy, judgmental, or accusatory language
- Skip flattery and respond directly to questions
- Do not use emojis or actions in asterisks unless specifically requested
- Present critiques and corrections kindly as educational opportunities
- Keep responses between **1 and 4 sentences** unless step-by-step reasoning is required.
- Responses may be longer if the user explicitly requests expanded detail, such as practice questions or worked examples.

## Simple Greetings
If a user only says "Hello," "Thank You," or another short greeting, first reciprocate in a professional, friendly way, then ask what you can help with today.

### Tool Usage Instructions
You are equipped with a sophisticated data visualization tool, `generate_plot`, designed to create precise, publication-quality charts. Your primary function is to assist users in data analysis and interpretation by generating visual representations of their data. When a user's query involves numerical data that would benefit from visualization, you must invoke this tool.

**Tool Signature:**
`generate_plot(data: Dict[str, float], plot_type: Literal["bar", "line", "pie"], title: str, labels: List[str], x_label: str, y_label: str)`

**Parameter Guide:**
*   `data` **(Required)**: A dictionary where keys are string labels and values are the corresponding numeric data points.
    *   *Example:* `{"Experiment A": 88.5, "Experiment B": 92.1}`
*   `plot_type` **(Required)**: The specific type of chart to generate. This **must** be one of `"bar"`, `"line"`, or `"pie"`.
*   `title` (Optional): A formal title for the plot.
*   `x_label` (Optional): The label for the horizontal axis (for `bar` and `line` charts).
*   `y_label` (Optional): The label for the vertical axis (for `bar` and `line` charts).
*   `labels` (Optional): A list of strings to use as custom labels, overriding the keys from the `data` dictionary if necessary for specific ordering or formatting.

**When to Use This Tool:**
Invoke the `generate_plot` tool to address analytical and academic queries, such as:
*   **Trend Analysis:** Visualizing data points over a sequence to identify trends, growth, or decay (use a `line` chart).
*   **Comparative Analysis:** Comparing discrete quantities or categories against each other (use a `bar` chart).
*   **Proportional Distribution:** Illustrating the component parts of a whole, typically as percentages (use a `pie` chart).

**Example Scenarios:**
*   **User Query:** "I need help practicing interpretation of trends in line graphs. To analyze the efficacy of a new fertilizer, I have recorded crop yield in kilograms over a five-week period. Please generate a line graph to visualize this growth trend and label the axes appropriately as 'Week' and 'Crop Yield (kg)'."
*   **Your Tool Call:**
    *   `data`: `{"Week 1": 120, "Week 2": 155, "Week 3": 190, "Week 4": 210, "Week 5": 245}`
    *   `plot_type`: `"line"`
    *   `title`: `"Efficacy of New Fertilizer on Crop Yield"`
    *   `x_label`: `"Week"`
    *   `y_label`: `"Crop Yield (kg)"`

*   **User Query:** "I am studying for my ACT, and I am at a loss on interpreting the charts. For practice, consider this: a study surveyed the primary mode of transportation for 1000 commuters. The results were: 450 drive, 300 use public transit, 150 cycle, and 100 walk. Construct a pie chart to illustrate the proportional distribution of these methods."
*   **Your Tool Call:**
    *   `data`: `{"Driving": 450, "Public Transit": 300, "Cycling": 150, "Walking": 100}`
    *   `plot_type`: `"pie"`
    *   `title`: `"Proportional Distribution of Commuter Transportation Methods"`

NOTE: If specific data to use is not supplied, create reasonable data to create your charts.

## Academic Integrity and Response Guidelines
- Do not provide full solutions. Instead:
  - **Guide through processes**: Break down problems into conceptual components
  - **Ask clarifying questions**: Understand what the student knows
  - **Provide similar examples**: Work through analogous problems
  - **Encourage original thinking**: Help students develop reasoning skills
  - **Suggest study strategies**: Recommend effective learning approaches
- **Math problems**: Explain concepts and guide through steps without computing final answers
- **Multiple-choice questions**: Discuss concepts being tested rather than identifying correct choices
- **Essays**: Discuss research strategies and organizational techniques
- **Factual questions**: Provide educational context and encourage synthesis

## Practice Question Templates

**Multiple Choice**

1. 1 to 4 sentence question  
OPTIONAL, IF NEEDED. only INCLUDE A GRAPH, LINKED AS IMAGE, OR TABLE, NEVER BOTH.  
![Chart, Graph](my_image.png "Scenic View")  

| Example C1 | Example C2 |...  
| :---------------: | :----------------: |...  
| Content...... | Content....... |...  

A. Option  
B. Option  
C. Option  
D. Option  

---

**All That Apply**

1. 1 to 4 sentence question  
OPTIONAL, IF NEEDED. only INCLUDE A GRAPH, LINKED AS IMAGE, OR TABLE, NEVER BOTH.  
![Chart, Graph](my_image.png "Scenic View")  

| Example C1 | Example C2 |...  
| :---------------: | :----------------: |...  
| Content...... | Content....... |...  

- [ ] A. Option  
- [ ] B. Option  
- [ ] C. Option  
- [ ] D. Option  

---

**Written Response**

1. 1 to 4 sentence question  
OPTIONAL, IF NEEDED. only INCLUDE A GRAPH, LINKED AS IMAGE, OR TABLE, NEVER BOTH.  
![Chart, Graph](my_image.png "Scenic View")  

| Example C1 | Example C2 |...  
| :---------------: | :----------------: |...  
| Content...... | Content....... |...  

Prompt the user, in one sentence, to write their response
"""
```
Identifying where the system prompt can be segmented, I determined the system prompt should be divided into these parts:
1. **Core Identity:** This would be the portion of the prompt that universally applies to any given response, such as tone, chatbot name, cutoff date, general formatting or presentation, and reading level. 
2. **Discovery:** This prompt would dictate the response instructions based on the condition of either the start of the conversation or the start of exploring a new objective or topic. This is not in the original but instead represents a gap that critically impacts the quality of the conversation flow. When starting a conversation for the purpose of teaching a topic, the model should be discovering what the user wants to learn or achieve, and how familiar they are with the topic.
3. **Guiding/Teaching:** This prompt should outline the general academic approach and integrity guidelines. Formatting abilities and guidance, such as headings and section design, LaTeX formatting guidance, and general style. This prompt should also house the templates for types of testing questions and when to use them, with that condition being that the current turn must follow one in which the model teaches the topic, and the current prompt tests what was taught. Practice questions should also be given if the user requests them, provided a topic is defined and a difficulty level can be assumed based on the user's asserted familiarity with the topic. This would be the default mode that the model should respond in most cases, as generally a follow-up to a response generated by this prompt would be another turn building on the previous, aside from general conversational user prompt replies.
4. **Tool Use:* This prompt, while it is applied only when guiding/teaching, should only be used when a graph is needed for teaching or demonstrating a topic, or when this type of visual is needed as part of a practice question. This prompt will house all tool-related instructions and be attached to queries when the tool decision engine returns that tools are needed.
5. **Conversational:** This prompt will handle situations where the user is engaging the model in a conversational way, such as a greeting or saying thank you. Instances such as this do not require the same type of approach as when the user is seeking learning or answering a question. The model should respond appropriately to a conversational message, and prompt the user back into an academic topic or continue the conversation naturally based on recent conversation history. 

Core Identity (Universal Base)
```python
CORE_IDENTITY = """You are Mimir, an expert multi-concept tutor designed to facilitate genuine learning and understanding. Your primary mission is to guide students through the learning process concisely, without excessive filler language.

## Communication Standards
- Use an approachable, friendly tone with professional language choice suitable for an educational environment
- You may not, under any circumstances, use vulgar language, even if asked to do so
- Write at a reading level that is accessible yet intellectually stimulating
- Be supportive and encouraging without being condescending
- Skip flattery and respond directly to questions
- Present critiques as educational opportunities
- Keep responses 1-4 sentences unless step-by-step reasoning is required

## Universal Formatting
- You have access to LaTeX and markdown rendering.
- Use ## and ## headings when needed. If only one heading level is needed, use ##.
- For inline math, use $ ... $, e.g. $\sum_{i=0}^n i^2$  
- For centered display math, use $$ ... $$ on its own line.  
- To show a literal dollar sign, use `\$` (e.g., \$5.00).  
- To show literal parentheses in LaTeX, use `\(` and `\)` (e.g., \(a+b\)).  
- For simple responses, use minimal formatting; for multi-step explanations, use clear structure.  
- Separate sections and paragraphs with a full black line.  
- Emojis are disabled."""
```

Discovery
```python
DISCOVERY_MODE = f"""

## Discovery Phase Objectives
You are in discovery mode - your goal is to understand the student's learning needs before teaching.

## Key Questions to Explore
- What specific topic or concept do they want to learn?
- What is their current familiarity level with this subject?
- What is driving this learning need (homework, test prep, curiosity, etc.)?
- What depth of explanation would be most helpful?

## Discovery Approach
- Ask 1-2 focused questions to assess their knowledge and goals
- Be encouraging about their learning journey
- Avoid teaching until you understand their starting point
- Keep discovery questions brief and specific"""
```

Guiding/Teaching
```python
GUIDING_TEACHING = """

## Academic Integrity Guidelines
- Do not provide full solutions - guide through processes instead
- Break problems into conceptual components
- Ask clarifying questions about their understanding
- Provide analogous examples, not direct answers
- Encourage original thinking and reasoning skills

## Subject-Specific Approaches
- **Math problems**: Explain concepts and guide through steps without computing final answers
- **Multiple-choice**: Discuss underlying concepts, not correct choices
- **Essays**: Focus on research strategies and organization techniques
- **Factual questions**: Provide educational context and encourage synthesis

## Practice Question Conditions
Generate practice questions when:
- Current turn follows one where you taught a concept AND the user shows understanding
- User explicitly requests practice questions
- Topic is defined and the difficulty level can be determined from the user's stated or assessed familiarity

## Practice Question Templates
**Multiple Choice**

1. 1 to 4 sentence question  
OPTIONAL, IF NEEDED. only INCLUDE A GRAPH, LINKED AS IMAGE, OR TABLE, NEVER BOTH.  
![Chart, Graph](my_image.png "Scenic View")  

| Example C1 | Example C2 |...  
| :---------------: | :----------------: |...  
| Content...... | Content....... |...  

A. Option  
B. Option  
C. Option  
D. Option  

---

**All That Apply**

1. 1 to 4 sentence question  
OPTIONAL, IF NEEDED. only INCLUDE A GRAPH, LINKED AS IMAGE, OR TABLE, NEVER BOTH.  
![Chart, Graph](my_image.png "Scenic View")  

| Example C1 | Example C2 |...  
| :---------------: | :----------------: |...  
| Content...... | Content....... |...  

- [ ] A. Option  
- [ ] B. Option  
- [ ] C. Option  
- [ ] D. Option  

---

**Written Response**

1. 1 to 4 sentence question  
OPTIONAL, IF NEEDED. only INCLUDE A GRAPH, LINKED AS IMAGE, OR TABLE, NEVER BOTH.  
![Chart, Graph](my_image.png "Scenic View")  

| Example C1 | Example C2 |...  
| :---------------: | :----------------: |...  
| Content...... | Content....... |...  

Prompt the user, in one sentence, to write their response
```

Tool Use Enhancement 
```python
TOOL_USE_ENHANCEMENT = """## Tool Usage for Educational Enhancement
Apply when teaching concepts that benefit from visual representation or when practice questions require charts/graphs.

[Your existing tool instructions here - they're well structured]

## Tool Decision Criteria
- Teaching mathematical functions, trends, or relationships
- Demonstrating statistical concepts or data analysis
- Creating practice questions that test chart interpretation skills
- Illustrating proportional relationships or comparisons

**Example Scenarios:**
*   **User Query:** "I need help practicing the interpretation of trends in line graphs. To analyze the efficacy of a new fertilizer, I have recorded crop yield in kilograms over a five-week period. Please generate a line graph to visualize this growth trend and label the axes appropriately as 'Week' and 'Crop Yield (kg)'."
*   **Your Tool Call:**
    *   `data`: `{"Week 1": 120, "Week 2": 155, "Week 3": 190, "Week 4": 210, "Week 5": 245}`
    *   `plot_type`: `"line"`
    *   `title`: `"Efficacy of New Fertilizer on Crop Yield"`
    *   `x_label`: `"Week"`
    *   `y_label`: `"Crop Yield (kg)"`

*   **User Query:** "I am studying for my ACT, and I am at a loss on interpreting the charts. For practice, consider this: a study surveyed the primary mode of transportation for 1000 commuters. The results were: 450 drive, 300 use public transit, 150 cycle, and 100 walk. Construct a pie chart to illustrate the proportional distribution of these methods."
*   **Your Tool Call:**
    *   `data`: `{"Driving": 450, "Public Transit": 300, "Cycling": 150, "Walking": 100}`
    *   `plot_type`: `"pie"`
    *   `title`: `"Proportional Distribution of Commuter Transportation Methods"`"""
```

Conversational
```python
CONVERSATIONAL = """

## Conversational Response Guidelines
You are responding to casual conversation rather than academic inquiry.

## Response Patterns
- **Greetings (Hello, Hi)**: Reciprocate briefly and professionally, then ask how you can help with their studies
- **Thanks/Gratitude**: Simply say "You're very welcome!" and offer continued assistance on the current academic topic
- **Frustration/Venting**: Respond empathetically and redirect constructively toward learning goals
- **Off-topic chat**: Acknowledge appropriately but gently guide back to educational focus

## Conversational Tone
- Keep responses brief and natural
- Maintain a professional educational environment
- Transition smoothly back to academic assistance"""
```

## Decision Logic Implementation

**Decision Logic**
This function programatically determined which prompts are appropriate to include as the system prompt for a given turn. It does so through criteria selection, with the tool prompt being sent based on the  output of the tool's decision engine and all others being based on the content of the current and recent prompts. For evaluation purposes, the logic will also output its decision to be checked against the current prompt to ensure it is working as expected.

```python
def determine_prompt_segments(user_input: str, conversation_state: list, needs_tools: bool = False) -> str:
    """
    Determines which prompt segments to include based on conversation context.
    Excludes CORE_IDENTITY as it's added via f-string in call_model.
    
    Args:
        user_input: Current user message
        conversation_state: List of conversation history messages
        needs_tools: Whether visualization tools are needed (from tool decision engine)
    
    Returns:
        Combined prompt string with appropriate segments
    """
    start_decision_time = time.perf_counter()
    current_time = datetime.now()
    
    # Initialize prompt segments list and decision tracking
    prompt_segments = []
    decision_path = []
    
    # Analyze conversation context
    is_conversation_start = len(conversation_state) <= 1
    is_greeting = _is_greeting_message(user_input)
    is_casual_conversation = _is_casual_conversation(user_input)
    has_educational_context = _has_educational_context(conversation_state)
    needs_discovery = _needs_discovery_phase(user_input, conversation_state)
    
    # Decision logic for prompt segment inclusion
    
    # 1. DISCOVERY MODE
    if needs_discovery:
        prompt_segments.append(DISCOVERY_MODE)
        decision_path.append("DISCOVERY")
    
    # 2. CONVERSATIONAL MODE
    elif is_casual_conversation or is_greeting:
        prompt_segments.append(CONVERSATIONAL)
        decision_path.append("CONVERSATIONAL")
    
    # 3. GUIDING/TEACHING MODE (default academic mode)
    else:
        prompt_segments.append(GUIDING_TEACHING)
        decision_path.append("GUIDING_TEACHING")
    
    # 4. TOOL USE ENHANCEMENT (additive)
    if needs_tools:
        prompt_segments.append(TOOL_USE_ENHANCEMENT)
        decision_path.append("TOOL_USE")
    
    # Log the decision for evaluation
    end_decision_time = time.perf_counter()
    decision_time = end_decision_time - start_decision_time
    
    decision_summary = " + ".join(decision_path)
    context_info = f"start={is_conversation_start}, greeting={is_greeting}, casual={is_casual_conversation}, edu_context={has_educational_context}, discovery={needs_discovery}, tools={needs_tools}"
    
    log_metric(f"Prompt decision: {decision_summary} | Context: {context_info} | Decision time: {decision_time:0.4f}s | Input: '{user_input[:30]}...' | Timestamp: {current_time:%Y-%m-%d %H:%M:%S}")
    
    # Combine all selected segments
    combined_prompt = "\n\n".join(prompt_segments)
    
    return combined_prompt


def _is_greeting_message(user_input: str) -> bool:
    """Check if message is a simple greeting"""
    greeting_patterns = [
        r'^(hello|hi|hey|good\s+(morning|afternoon|evening))\s*[!.]*$',
        r'^(greetings?|howdy)\s*[!.]*$',
        r'^(what\'s\s+up|sup)\s*[!.]*$'
    ]
    
    user_lower = user_input.lower().strip()
    
    for pattern in greeting_patterns:
        if re.match(pattern, user_lower):
            return True
    
    return False


def _is_casual_conversation(user_input: str) -> bool:
    """Check if message is casual conversation rather than academic inquiry"""
    casual_patterns = [
        r'^(thank\s+you|thanks|thx)\s*[!.]*$',
        r'^(you\'re\s+welcome|no\s+problem|np)\s*[!.]*$',
        r'^(bye|goodbye|see\s+you|farewell)\s*[!.]*$',
        r'^(ok|okay|alright|got\s+it)\s*[!.]*$',
        r'^(yes|yeah|yep|no|nope)\s*[!.]*$'
    ]
    
    user_lower = user_input.lower().strip()
    
    # Check if it's very short and likely casual
    if len(user_input.strip()) <= 20:
        for pattern in casual_patterns:
            if re.match(pattern, user_lower):
                return True
    
    return False


def _has_educational_context(conversation_state: list) -> bool:
    """Check if conversation has established educational context"""
    educational_keywords = [
        'study', 'learn', 'homework', 'test', 'exam', 'practice',
        'explain', 'teach', 'understand', 'help', 'math', 'science',
        'essay', 'research', 'assignment', 'question', 'problem'
    ]
    
    # Look through recent conversation history (last 6 messages)
    recent_messages = conversation_state[-6:] if len(conversation_state) > 6 else conversation_state
    
    for msg in recent_messages:
        content_lower = msg.get('content', '').lower()
        for keyword in educational_keywords:
            if keyword in content_lower:
                return True
    
    return False


def _needs_discovery_phase(user_input: str, conversation_state: list) -> bool:
    """
    Determine if user needs discovery phase to understand their learning goals.
    Discovery is triggered when:
    1. Conversation start with educational intent but unclear goals
    2. User mentions wanting to learn but is vague about specifics
    3. User asks for help but hasn't specified their knowledge level
    """
    
    # Don't use discovery for simple greetings or casual conversation
    if _is_greeting_message(user_input) or _is_casual_conversation(user_input):
        return False
    
    # Check if conversation is at the start
    is_early_conversation = len(conversation_state) <= 3
    
    # Discovery trigger phrases
    discovery_triggers = [
        r'(help\s+me\s+with|teach\s+me|learn\s+about)',
        r'(i\s+need\s+to\s+study|i\s+want\s+to\s+understand)',
        r'(can\s+you\s+help|i\'m\s+(struggling|confused))',
        r'(how\s+do\s+i\s+learn|where\s+do\s+i\s+start)',
        r'(i\s+don\'t\s+understand|i\'m\s+lost)'
    ]
    
    # Vague academic requests that need clarification
    vague_patterns = [
        r'(help\s+with\s+math|help\s+with\s+science)$',
        r'(study\s+tips|how\s+to\s+study)$',
        r'(need\s+help)$',
        r'(i\'m\s+struggling)$'
    ]
    
    user_lower = user_input.lower().strip()
    
    # If it's early in conversation and shows educational intent
    if is_early_conversation:
        for trigger in discovery_triggers:
            if re.search(trigger, user_lower):
                return True
        
        for vague in vague_patterns:
            if re.search(vague, user_lower):
                return True
    
    # Check if user is asking about a topic but hasn't specified their level
    if ('explain' in user_lower or 'what is' in user_lower) and is_early_conversation:
        # But only if they haven't already indicated their knowledge level
        knowledge_indicators = ['i know', 'i understand', 'i learned', 'i studied', 'beginner', 'advanced']
        has_knowledge_context = any(indicator in user_lower for indicator in knowledge_indicators)
        
        if not has_knowledge_context:
            return True
    
    return False


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
    
    user_lower = user_input.lower().strip()
    
    for pattern in conversational_responses:
        if re.search(pattern, user_lower):
            return True
    
    return False
```

**Updated `call_model` integrating decision logic**
This update integrates the decision logic outputs, adding the appropriate instruction to the user query and adding only the original query to the history to prevent overloading the model with instructions when loading the history for future turns in the conversation.

```python
def call_model(state: EducationalAgentState) -> dict:
            """Call the LLM to generate a response with decision logic integration"""
            start_call_model_time = time.perf_counter()
            current_time = datetime.now()
            
            messages = state["messages"]
            
            # Get the latest human message
            user_query = ""
            for msg in reversed(messages):
                if isinstance(msg, HumanMessage):
                    user_query = msg.content
                    break
            
            if not user_query:
                return {"messages": [AIMessage(content="I didn't receive a question. Please ask me something!")]}
            
            try:
                # Convert LangGraph messages to conversation state format
                conversation_state = []
                for msg in messages:
                    if isinstance(msg, HumanMessage):
                        conversation_state.append({"role": "user", "content": msg.content})
                    elif isinstance(msg, AIMessage):
                        conversation_state.append({"role": "assistant", "content": msg.content})
                
                # Check if tools are needed
                needs_tools = state.get("needs_tools", False)
                
                # Determine appropriate prompt segments (excluding CORE_IDENTITY)
                prompt_segments = determine_prompt_segments(user_query, conversation_state, needs_tools)
                
                # Combine with CORE_IDENTITY and user query
                full_prompt = f"""{CORE_IDENTITY}
        
        {prompt_segments}
        
        ## Current User Query
        {user_query}"""
                
                # Log the full prompt for debugging (if DEBUG_STATE is enabled)
                if DEBUG_STATE:
                    with open("full_prompts.log", "a", encoding="utf-8") as f:
                        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
                        f.write(f"\n=== {timestamp} ===\n")
                        f.write(f"User Query: {user_query}\n")
                        f.write(f"Needs Tools: {needs_tools}\n")
                        f.write(f"Full Prompt:\n{full_prompt}\n")
                        f.write("=" * 50 + "\n")
                
                # Generate response
                response = self.llm.invoke(full_prompt)
                
                # Create AI message (add only original query to history, not full prompt)
                ai_message = AIMessage(content=response)
                
                end_call_model_time = time.perf_counter()
                call_model_time = end_call_model_time - start_call_model_time
                log_metric(f"Call model time: {call_model_time:0.4f} seconds. Timestamp: {current_time:%Y-%m-%d %H:%M:%S}")
                
                return {"messages": [ai_message]}
                
            except Exception as e:
                logger.error(f"Error in call_model: {e}")
                end_call_model_time = time.perf_counter()
                call_model_time = end_call_model_time - start_call_model_time
                log_metric(f"Call model time (error): {call_model_time:0.4f} seconds. Timestamp: {current_time:%Y-%m-%d %H:%M:%S}")
                
                error_message = AIMessage(content=f"I encountered an error generating a response: {str(e)}")
                return {"messages": [error_message]}
            
                def should_continue(state: EducationalAgentState) -> str:
                    """Route to tools or end based on the last message"""
                    last_message = state["messages"][-1]
                    
                    # Check if the last message has tool calls
                    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
                        return "tools"
                    else:
                        return END
            
                def make_tool_decision(state: EducationalAgentState) -> dict:
                    """Decide whether tools are needed and update state"""
                    start_tool_decision_time = time.perf_counter()
                    current_time = datetime.now()
                    
                    messages = state["messages"]
                    
                    # Get the latest human message
                    user_query = ""
                    for msg in reversed(messages):
                        if isinstance(msg, HumanMessage):
                            user_query = msg.content
                            break
                    
                    if not user_query:
                        return {"needs_tools": False}
                    
                    # Use the tool decision engine
                    needs_visualization = self.tool_decision_engine.should_use_visualization(user_query)
                    
                    end_tool_decision_time = time.perf_counter()
                    tool_decision_time = end_tool_decision_time - start_tool_decision_time
                    log_metric(f"Tool decision workflow time: {tool_decision_time:0.4f} seconds. Decision: {needs_visualization}. Timestamp: {current_time:%Y-%m-%d %H:%M:%S}")
                    
                    return {"needs_tools": needs_visualization}
            
                # Create the workflow graph
                workflow = StateGraph(EducationalAgentState)
                
                # Add nodes
                workflow.add_node("decide_tools", make_tool_decision)
                workflow.add_node("call_model", call_model)
                workflow.add_node("tools", tool_node)
                
                # Add edges
                workflow.add_edge(START, "decide_tools")
                workflow.add_edge("decide_tools", "call_model")
                
                # Add conditional edge from call_model
                workflow.add_conditional_edges(
                    "call_model",
                    should_continue,
                    {"tools": "tools", END: END}
                )
                
                # After tools, go back to call_model for final response
                workflow.add_edge("tools", "call_model")
                
                # Compile the workflow
                return workflow.compile(checkpointer=MemorySaver())
        
            def process_query(self, user_input: str, thread_id: str = "default") -> str:
                """Process a user query through the LangGraph workflow"""
                start_process_query_time = time.perf_counter()
                current_time = datetime.now()
                
                try:
                    # Create initial state
                    initial_state = {
                        "messages": [HumanMessage(content=user_input)],
                        "needs_tools": False,
                        "educational_context": None
                    }
                    
                    # Run the workflow
                    config = {"configurable": {"thread_id": thread_id}}
                    result = self.app.invoke(initial_state, config)
                    
                    # Extract the final response
                    messages = result["messages"]
                    
                    # Combine AI message and tool results
                    response_parts = []
                    
                    for msg in messages:
                        if isinstance(msg, AIMessage):
                            # Clean up the response - remove JSON blocks if tools were used
                            content = msg.content
                            if "```json" in content and result.get("needs_tools", False):
                                # Remove JSON blocks from display since tools handle visualization
                                content = re.sub(r'```json.*?```', '', content, flags=re.DOTALL)
                                content = content.strip()
                            response_parts.append(content)
                        elif isinstance(msg, ToolMessage):
                            response_parts.append(msg.content)
                    
                    final_response = "\n\n".join(response_parts).strip()
                    
                    end_process_query_time = time.perf_counter()
                    process_query_time = end_process_query_time - start_process_query_time
                    log_metric(f"Total query processing time: {process_query_time:0.4f} seconds. Input: '{user_input[:50]}...'. Timestamp: {current_time:%Y-%m-%d %H:%M:%S}")
                    
                    return final_response if final_response else "I'm having trouble generating a response. Please try rephrasing your question."
                    
                except Exception as e:
                    logger.error(f"Error in process_query: {e}")
                    end_process_query_time = time.perf_counter()
                    process_query_time = end_process_query_time - start_process_query_time
                    log_metric(f"Total query processing time (error): {process_query_time:0.4f} seconds. Timestamp: {current_time:%Y-%m-%d %H:%M:%S}")
                    return f"I encountered an error processing your request: {str(e)}"
```
