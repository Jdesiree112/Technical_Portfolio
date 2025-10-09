## User Message Path
This document outlines the path of the user's message throughout the full process, including a flow diagram and key transformations.

### User Message Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USER TYPES MESSAGE                          │
│                    "Hello" in Gradio textbox                        │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  GRADIO EVENT: textbox.submit                                       │
│  ├─> add_user_message_persistent(message, chat_history, conv_state) │
│  │   ├─> global_state_manager.get_conversation_state()              │
│  │   ├─> spellchecker.correct_text(message)                         │
│  │   ├─> conversation_state.append({"role": "user", "content": msg})│
│  │   └─> global_state_manager.update_conversation_state()           │
│  └─> Returns: ("", updated_chat_history, updated_conversation_state)│
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  GRADIO EVENT: .then chain triggered                                │
│  ├─> add_loading_animation_persistent(chat_history, conv_state)     │
│  │   ├─> get_loading_animation_base64() - loads GIF                 │
│  │   ├─> chat_history.append({"role": "assistant", "content": GIF}) │
│  │   └─> global_state_manager.update_conversation_state()           │
│  └─> User sees animated "thinking" GIF                              │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  GRADIO EVENT: .then chain continues                                │
│  └─> generate_response_persistent(chat_history, conversation_state) │
│      └─> ENTERS MAIN PROCESSING                                     │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  generate_response_persistent()                                     │
│  ├─> _ensure_agent() - loads agent if needed                        │
│  ├─> global_state_manager.get_conversation_state()                  │
│  ├─> Extract last user message from conversation_state              │
│  ├─> remove_loading_animations(chat_history)                        │
│  └─> FOR EACH CHUNK IN agent.stream_query(last_user_message):       │
│      ├─> Update chat_history[-1]["content"] = chunk                 │
│      └─> yield chat_history, conversation_state (UPDATES UI)        │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Educational_Agent.stream_query(user_input, thread_id)               │
│  ├─> _ensure_all_components() - loads lazy components                │
│  │   ├─> tool_decision_engine (TinyLlama model)                      │
│  │   ├─> post_processor (ResponsePostProcessor)                      │
│  │   ├─> classifier (ML prompt classifier)                           │
│  │   └─> app (LangGraph workflow)                                    │
│  │                                                                   │
│  ├─> Create initial_state:                                           │
│  │   {                                                               │
│  │     "messages": [HumanMessage(content="Hello")],                  │
│  │     "needs_tools": False,                                         │
│  │     "educational_context": None                                   │
│  │   }                                                               │
│  │                                                                   │
│  └─> result = self.app.invoke(initial_state, config) ────┐           │
└───────────────────────────────────────────────────────────┼──────────┘
                                                            │
                                                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  LANGGRAPH WORKFLOW EXECUTION                                       │
│  (StateGraph with MemorySaver checkpoint)                           │
│                                                                     │
│  START                                                              │
│    │                                                                │
│    ▼                                                                │
│  ┌──────────────────────────────────────────┐                       │
│  │ NODE: decide_tools                       │                       │
│  │ (make_tool_decision function)            │                       │
│  ├──────────────────────────────────────────┤                       │
│  │ ├─> Extract user query from messages     │                       │
│  │ ├─> tool_decision_engine.should_use_     │                       │
│  │ │    visualization(query)                │                       │
│  │ │    └─> TinyLlama inference (5 tokens)  │                       │
│  │ │    └─> Returns: YES/NO                 │                       │
│  │ └─> Returns: {"needs_tools": True/False} │                       │
│  └──────────────────────────────────────────┘                       │
│    │                                                                │
│    ▼                                                                │
│  ┌──────────────────────────────────────────┐                       │
│  │ NODE: call_model                         │                       │
│  │ (Main LLM inference)                     │                       │
│  ├──────────────────────────────────────────┤                       │
│  │ ┌─ STAGE 0: PROMPT CONSTRUCTION ───────┐ │                       │
│  │ │ ├─> Extract user_query from messages  │ │                      │
│  │ │ ├─> _determine_prompt_segments_ml()   │ │                      │
│  │ │ │   ├─> _get_conversation_metadata()  │ │                      │
│  │ │ │   ├─> _calculate_classifier_features│ │                      │
│  │ │ │   ├─> make_classification_with_     │ │                      │
│  │ │ │   │    tracking()                   │ │                      │
│  │ │ │   │   └─> ML classifier predicts:   │ │                      │
│  │ │ │   │       • use_discovery_mode      │ │                      │
│  │ │ │   │       • use_conversational      │ │                      │
│  │ │ │   │       • use_guiding_teaching    │ │                      │
│  │ │ │   │       • use_tool_enhancement    │ │                      │
│  │ │ │   └─> Returns: prompt_segments      │ │                      │
│  │ │ │                                     │ │                      │
│  │ │ ├─> Build complete_prompt:            │ │                      │
│  │ │ │   CORE_IDENTITY +                   │ │                      │
│  │ │ │   prompt_segments +                 │ │                      │
│  │ │ │   recent_history +                  │ │                      │
│  │ │ │   user_query +                      │ │                      │
│  │ │ │   knowledge_cutoff                  │ │                      │
│  │ │ └─> complete_prompt (string)          │ │                      │
│  │ └───────────────────────────────────────┘ │                      │
│  │                                           │                      │
│  │ ┌─ STAGE 1: LLM INFERENCE ──────────────┐ │                      │
│  │ │ ├─> self.llm.invoke(complete_prompt)  │ │                      │
│  │ │ │   └─> Phi3MiniEducationalLLM.invoke │ │                      │
│  │ │ │       ├─> _format_chat_template()   │ │                      │
│  │ │ │       ├─> tokenizer(text)           │ │                      │
│  │ │ │       ├─> @spaces.GPU decorator     │ │                      │
│  │ │ │       │   activates GPU             │ │                      │
│  │ │ │       ├─> _load_and_prepare_model() │ │                      │
│  │ │ │       │   └─> Loads Phi-3 w/ PEFT   │ │                      │
│  │ │ │       ├─> model.generate()          │ │                      │
│  │ │ │       │   (max_new_tokens=350)      │ │                      │
│  │ │ │       └─> tokenizer.decode()        │ │                      │
│  │ │ │                                     │ │                      │
│  │ │ └─> response (string, 75 chars) ────┐ │ │                      │
│  │ └─────────────────────────────────────┼─┘ │                      │
│  │                                       │   │                      │
│  │ ┌─ STAGE 2: QUALITY TRACKING ─────────┼─┐ │                      │
│  │ │ ├─> evaluate_educational_quality_   │ │ │                      │
│  │ │ │    with_tracking()                │ │ │                      │
│  │ │ │   ├─> Calculate educational score │ │ │                      │
│  │ │ │   └─> global_state_manager.       │ │ │                      │
│  │ │ │        add_educational_quality_   │ │ │                      │
│  │ │ │        score()                    │ │ │                      │
│  │ │ └─> Metrics logged                  │ │ │                      │
│  │ └─────────────────────────────────────┘ │ │                      │
│  │                                        │ │                       │
│  │ ┌─ STAGE 3: CREATE AI MESSAGE ─────────┼─┐│                      │
│  │ │ ├─> ai_message = AIMessage(          │ ││                      │
│  │ │ │       content=response)            │ ││                      │
│  │ │ │   (LangChain message object)       │ ││                      │
│  │ │ └─> ai_message created ──────────────┼─┼┘                      │
│  │ └──────────────────────────────────────┘ │                       │
│  │                                          │                       │
│  │ ┌─ STAGE 4: RETURN ────────────────────┐ │                       │
│  │ │ └─> return {"messages": [ai_message]} │                        │
│  │ └──────────────────────────────────────┘ │                       │
│  └──────────────────┬───────────────────────┘                       │
│                     │                                               │
│                     ▼                                               │
│  ┌──────────────────────────────────────────┐                       │
│  │ NODE: should_continue (conditional)      │                       │
│  ├──────────────────────────────────────────┤                       │
│  │ ├─> Check ai_message.tool_calls          │                       │
│  │ ├─> If has tool_calls → "tools"          │                       │
│  │ └─> If no tool_calls → END               │                       │
│  └──────────────────┬───────────────────────┘                       │
│                     │                                               │
│                     ▼                                               │
│         ┌───────────────────────┐                                   │
│         │ Has tool_calls?       │                                   │
│         └──────┬────────┬───────┘                                   │
│                │ YES    │ NO                                        │
│                ▼        ▼                                           │
│         ┌──────────┐  END                                           │
│         │ NODE:    │   │                                            │
│         │ tools    │   │                                            │
│         │ (ToolNode│   │                                            │
│         └──────┬───┘   │                                            │
│                │       │                                            │
│                ├───────┘                                            │
│                │ (loops back to call_model)                         │
│                │                                                    │
│                ▼                                                    │
│               END                                                   │
│                │                                                    │
└────────────────┼───────────────────────────────────────────────────┘
                 │
                 │ result = {
                 │   "messages": [
                 │     HumanMessage(content="Hello"),
                 │     AIMessage(content="<75 char response>")
                 │   ]
                 │ }
                 │
                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  BACK IN: Educational_Agent.stream_query()                          │
│  ├─ STAGE 7: EXTRACT RESPONSE FROM WORKFLOW ───────────────────────┐│
│  │ ├─> messages = result["messages"]                               ││
│  │ ├─> Loop through messages:                                      ││
│  │ │   ├─> If AIMessage: extract content                           ││
│  │ │   │   ├─> Clean JSON blocks (if tools used)                   ││
│  │ │   │   └─> response_parts.append(content)                      ││
│  │ │   └─> If ToolMessage: append content                          ││
│  │ └─> raw_response = "\n\n".join(response_parts).strip()          ││
│  └────────────────────────────────────────────────────────────────┘│
│                                                                    │
│  ├─ STAGE 8-9: POST-PROCESSING & STREAMING ───────────────────────┐│
│  │ ├─> post_processor.process_and_stream_response(                ││
│  │ │       raw_response, user_input)                              ││
│  │ │   │                                                          ││
│  │ │   ├─ STAGE 10: POST-PROCESSING ──────────────────────────┐   ││
│  │ │   │ ├─> process_response(raw_response, user_query)       │   ││
│  │ │   │ │   ├─> _enhanced_token_cleanup()                    │   ││
│  │ │   │ │   ├─> _remove_repetition()                         │   ││
│  │ │   │ │   ├─> _truncate_intelligently()                    │   ││
│  │ │   │ │   ├─> _validate_educational_content()              │   ││
│  │ │   │ │   └─> _enhance_readability()                       │   ││
│  │ │   │ └─> processed_response (cleaned string)              │   ││
│  │ │   └────────────────────────────────────────────────────────┘ ││
│  │ │                                                              ││
│  │ │   ├─ STAGE 11: WORD-BY-WORD STREAMING ───────────────────┐   ││
│  │ │   │ ├─> words = processed_response.split()               │   ││
│  │ │   │ ├─> FOR EACH word:                                   │   ││
│  │ │   │ │   ├─> current_output += word + " "                 │   ││
│  │ │   │ │   ├─> yield current_output ──────────────────────┐ │   ││
│  │ │   │ │   └─> time.sleep(0.015) # 15ms delay             │ │   ││
│  │ │   │ └─> Streaming complete                             │ │   ││
│  │ │   └────────────────────────────────────────────────────┘ │   ││
│  │ │                                                          │   ││
│  │ └─> FOR EACH chunk yielded: ◄─────────────────────────────┘    ││
│  │     └─> yield chunk (back to generate_response_persistent) ────┤│
│  └────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────┬──────────────────────────────────┘
                                  │
                                  │ Yields stream back...
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  BACK IN: generate_response_persistent()                            │
│  ├─> FOR EACH chunk from agent.stream_query():                      │
│  │   ├─> Update chat_history[-1]["content"] = chunk                 │
│  │   ├─> global_state_manager.update_conversation_state()           │
│  │   └─> yield chat_history, conversation_state ────────────────┐   │
│  │                                                              │   │
│  ├─> After streaming complete:                                  │   │
│  │   ├─> Apply post_processor.process_response()                │   │
│  │   ├─> conversation_state.append(final_response)              │   │
│  │   └─> global_state_manager.update_conversation_state()       │   │
│  │                                                              │   │
│  └─> Final yield: (chat_history, conversation_state) ───────────┼───┤
└─────────────────────────────────────────────────────────────────┼───┘
                                                                   │
                                                                   │
                                                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│  GRADIO UI UPDATE                                                   │
│  ├─> Chatbot component receives updated chat_history                │
│  ├─> Renders messages in chat interface                             │
│  │   ├─> User message: "Hello"                                      │
│  │   └─> Assistant message: "<final processed response>"            │
│  │                                                                  │
│  └─> USER SEES RESPONSE (word-by-word animation)                    │
└─────────────────────────────────────────────────────────────────────┘
```
### Key Data Transformations

```
User Input (str)
  "Hello"
    ↓
HumanMessage(content="Hello")  [LangChain format]
    ↓
StateGraph processes
    ↓
LLM generates response (str)
  "Hello! I'm Mimir, your educational AI assistant..."
    ↓
AIMessage(content="Hello! I'm Mimir...")  [LangChain format]
    ↓
Extracted back to string
  "Hello! I'm Mimir..."
    ↓
Post-processed (cleaned, formatted)
  "Hello! I'm Mimir, your educational AI assistant..."
    ↓
Streamed word-by-word (generator)
  "Hello" → "Hello! I'm" → "Hello! I'm Mimir" → ...
    ↓
Gradio chat_history update
  [{"role": "assistant", "content": "Hello! I'm Mimir..."}]
    ↓
User sees response
```
