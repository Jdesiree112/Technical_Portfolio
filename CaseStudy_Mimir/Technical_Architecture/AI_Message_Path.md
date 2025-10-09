## AI Message Path

This document outlines the path of the `ai_message` variable.

### Flow Diagram
```
USER INPUT → LangGraph Workflow → call_model() → LLM → RESPONSE
                                                          ↓
                    STAGE 1: Raw string response received from LLM
                              (75 characters)
                                                          ↓
                    STAGE 2: Educational quality tracking
                              (logs metrics to global state)
                                                          ↓
                    STAGE 3: AIMessage object created
                              response (str) → ai_message (AIMessage)
                                                          ↓
                    STAGE 4: Wrapped in return dict
                              {"messages": [ai_message]}
                                                          ↓
                    STAGE 5: Return to LangGraph workflow
                                                          ↓
                    STAGE 6: should_continue() routing
                              (checks for tool_calls, routes to END)
                                                          ↓
                    STAGE 7: Extract from workflow result
                              result["messages"] → response_parts[]
                                                          ↓
                    STAGE 8: Clean JSON blocks (if tools used)
                              "\n\n".join(response_parts)
                                                          ↓
                    STAGE 9: Send to post-processor
                                                          ↓
                    STAGE 10: Post-processing pipeline
                              - Token cleanup
                              - Remove repetition
                              - Intelligent truncation
                              - Validate educational content
                              - Enhance readability
                                                          ↓
                    STAGE 11: Stream word-by-word to UI
                              yield chunk by chunk
                                                          ↓
                              USER SEES RESPONSE
```
