# Mimir Educational AI Assistant - API Integration Guide

**Version:** 1.0  
**Created:** September 2025  
**Platform:** Hugging Face Spaces (Gradio)  
**Space:** `jdesiree/Mimir`

## Overview

Mimir is an educational AI assistant hosted on Hugging Face Spaces built with Gradio. This API documentation covers the 7 available endpoints for programmatic integration with the conversational interface.

## Quick Start

### Installation

```bash
pip install gradio_client
```

### Basic Connection

```python
from gradio_client import Client

client = Client("jdesiree/Mimir")
```

## API Endpoints

### Message Handling

#### `/add_user_message`
Adds user message to state and displays immediately.

**Parameters:**
- `message` (str, required): The user input message
- `chat_history` (list, optional, default: []): Current chat history

**Returns:**
- Tuple of 2 elements:
  - [0] str: Updated message value 
  - [1] list: Updated chat history

**Example:**
```python
result = client.predict(
    message="Hello!!",
    chat_history=[],
    api_name="/add_user_message"
)
message_value, updated_history = result
```

#### `/add_user_message_1`
Duplicate endpoint - same functionality as `/add_user_message`.

### Response Generation

#### `/generate_response`
Generates streaming response from the agent with post-processing enhancements.

**Parameters:**
- `chat_history` (list, optional, default: []): Current conversation history

**Returns:**
- list: Updated chat history with generated response

**Example:**
```python
response = client.predict(
    chat_history=[],
    api_name="/generate_response"
)
```

#### `/generate_response_1`
Duplicate endpoint - same functionality as `/generate_response`.

### UI Enhancement

#### `/add_loading_animation`
Adds loading animation to chat display without modifying conversation state.

**Parameters:**
- `chat_history` (list, optional, default: []): Current chat display state

**Returns:**
- list: Chat history with loading animation

**Example:**
```python
loading_display = client.predict(
    chat_history=[],
    api_name="/add_loading_animation"
)
```

#### `/add_loading_animation_1`
Duplicate endpoint - same functionality as `/add_loading_animation`.

### Session Management

#### `/reset_conversation`
Resets both chat display and conversation state.

**Parameters:**
- None

**Returns:**
- list: Empty chat history

**Example:**
```python
reset_state = client.predict(api_name="/reset_conversation")
```

## Chat History Structure

The chat_history parameter uses this structure:

```python
list[dict(
    role: str,
    metadata: dict(
        title: str,
        id: int | str,
        parent_id: int | str,
        log: str,
        duration: float,
        status: Literal['pending', 'done']
    ) | None,
    content: str | dict(file: filepath, alt_text: str | None) | dict(
        component: str,
        value: Any,
        constructor_args: dict(str, Any),
        props: dict(str, Any)
    ),
    options: list[dict(value: str, label: str)] | None
)]
```

## Basic Integration Patterns

### Simple Conversation Flow

```python
from gradio_client import Client

client = Client("jdesiree/Mimir")

# Start fresh conversation
client.predict(api_name="/reset_conversation")

# Add user message
result = client.predict(
    message="Explain photosynthesis",
    chat_history=[],
    api_name="/add_user_message"
)

# Generate AI response
response = client.predict(
    chat_history=result[1],
    api_name="/generate_response"
)

print(response)
```

### Conversation with Loading States

```python
# Add user message
result = client.predict(
    message="What is calculus?",
    chat_history=[],
    api_name="/add_user_message"
)

# Show loading animation
loading_state = client.predict(
    chat_history=result[1],
    api_name="/add_loading_animation"
)

# Generate response
final_response = client.predict(
    chat_history=result[1],  # Use original history, not loading state
    api_name="/generate_response"
)
```

### Multi-turn Conversation

```python
chat_history = []

# First exchange
result1 = client.predict(
    message="I need help with algebra",
    chat_history=chat_history,
    api_name="/add_user_message"
)

response1 = client.predict(
    chat_history=result1[1],
    api_name="/generate_response"
)

# Continue conversation
result2 = client.predict(
    message="Can you give me an example?",
    chat_history=response1[0],
    api_name="/add_user_message"
)

response2 = client.predict(
    chat_history=result2[1],
    api_name="/generate_response"
)
```

## Error Handling

```python
def safe_api_call(client, message, chat_history=None):
    """Wrapper with basic error handling"""
    if chat_history is None:
        chat_history = []
    
    try:
        result = client.predict(
            message=message,
            chat_history=chat_history,
            api_name="/add_user_message"
        )
        return result
    except Exception as e:
        print(f"API call failed: {e}")
        return None, chat_history
```

## Notes

- The API includes duplicate endpoints (`/add_user_message_1`, `/generate_response_1`, `/add_loading_animation_1`) that provide identical functionality
- Loading animations affect display only, not conversation state
- Chat history maintains conversation context across API calls
- All endpoints return structured data following Gradio's component specifications

## Live Interface

**Hugging Face Space:** https://jdesiree-mimir.hf.space
