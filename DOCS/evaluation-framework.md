# Evaluation Framework

## Document Summary

This document demonstrates my approach to systematic prompt evaluation using the OpenAI Evals API. It presents a complete workflow for testing and refining classification prompts in a customer service context, showcasing the iterative process I employ to identify weaknesses in prompt design and implement targeted improvements. The example illustrates my methodology for creating structured test cases, anticipating edge cases, and applying quantitative assessment to guide prompt optimization decisions.

## OpenAI Evals API Implementation Example

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
