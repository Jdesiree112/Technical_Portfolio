## Prompt Library

This is a collection of examples of prompts, written for a variety of purposes. The main objective of this document is to demonstrate prompting strategies and patterns to show competency in creating effective prompts. Some use cases to be demonstrated are: targeted behavior evaluation, establishing response format, tuning model tone and persona, defining safeguards against misuse, or utility-based instructions. Prompting patterns may be combined to achieve more effective system prompts. A section will be dedicated to exploring OpenAI API evals, where I will include test user input to demonstrate the use of this tool in performance evaluations. To supplement this, I will also include a section to cover the strengths and ideal use of different foundation models.

### OpenAI API for Evals

#### Example One
**Establishing a Categorization Function**

Here I import OpenAI and define required variables using Python. The model was given a system prompt, saved under the variable `instructions` here, that defines a simple categorization task. The model is to read the incoming user message, and assign it to one of the provided applicable categories or `Undefined/General` if none fit.

For this example, I selected ChatGPT to be the model because it is an OpenAI foundation model that is known for performing well in text categorization tasks.

```
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

System Prompt: You are a customer service professional skilled in identifying client needs. Given a user message, categorize it using one of these options: "Delivery Issue(s)," "Product Issue(s)," "Subscription Modify/Cancel," "Account Issue(s)," or "Undefined/General." Respond with only one category that best matches the user message. Use "Undefined/General" only when no other options apply.

**Setting Up Evaluation**

```
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

In this setup, the operation is checking for an exact match (eq).

**Defining Test Cases**

Using the setup evaluation and a JSONL file of test cases, the model can be evaluated on its performance with the system prompt. This allows for informed adjustments to be made to the system prompt based on what areas the model underperforms in. In the example established here, I would be most interested in checking that the model does not apply the `Undefined/General` category inappropriately, or fail to apply it when it would be more fitting than a loosely fitting category.

```
{"item": {"user_message": "I got an email that my account was locked from too many login attempts.", "correct_label": "Account Issue(s)"}}
{"item": {"user_message": "My order hasn't arrived yet and it's been 2 weeks", "correct_label": "Delivery Issue(s)"}}
{"item": {"user_message": "The product I received is damaged", "correct_label": "Product Issue(s)"}}
{"item": {"user_message": "I want to cancel my subscription, this costs too much.", "correct_label": "Subscription Modify/Cancel"}}
{"item": {"user_message": "Customer service number please", "correct_label": "Undefined/General"}}
{"item": {"user_message": "Update email address", "correct_label": "Undefined/General"}}
```

#### Example Two
**Shipment Updates**

The model was given a system prompt, saved under the variable `instructions` here, that defines a content extraction and summarization task, with some presentation parameters and general tone guidance. Formatting specifications for dates and times are included to make the output more compatible with programmatic evaluations using the OpenAI API.

```
from openai import OpenAI
client = OpenAI()

instructions = """
You are a shipment tracking expert. You review emails on shipments or deliveries, and generate a friendly message to the user. You inform the user of any updates received over email in a concise message, focusing on providing the following information: sender, tracking number, expected delivery date, and order status. If one of the listed items is not included in the email, omit it from your summary message. The message should not exceed four sentences and should be direct. Using Markdown, apply bolding to key terms. Format dates in a MONTH DD, YYYY format, and times as 00:00 XM.
"""
email_text = " "

response = client.responses.create(
    model="gpt-4.1",
    input=[
        {"role": "developer", "content": instructions},
        {"role": "user", "content": email_text},
    ],
)

print(response.output_text)

```

**Setting Up Evaluation**

```
from openai import OpenAI
client = OpenAI()
eval_obj = client.evals.create(
    name="Shipment Summary Key Terms Check",
    data_source_config={
        "type": "custom",
        "item_schema": {
            "type": "object",
            "properties": {
                "email_text": { 
                    "type": "string",
                    "description": "Email Text Content"
                },
                "expected_key_terms": { 
                    "type": "array",
                    "items": {"type": "string"},
                    "description": "Key Terms Expected in Summary"
                }
            },
            "required": ["email_text", "expected_key_terms"]
        },
        "include_sample_schema": True
    },
    testing_criteria=[
        {
            "type": "string_check",
            "name": "Contains Expected Key Terms",
            "input": "{{ sample.output_text }}",
            "operation": "contains",
            "reference": "{{ item.expected_key_terms }}"
        }
    ]
)
print(f"Created eval with ID: {eval_obj.id}")
```
This example uses the `contains` operation to check that key terms are present in model responses rather than an exact match.

**Defining Test Cases**

```
{"item": {"email_text": "Hello John, Your Amazon order has shipped! We're writing to let you know that your recent order has been dispatched from our fulfillment center. Order #112-7834561-9876543 Tracking number: 1Z879E03YW91234567 Shipped via: UPS Expected delivery: Wednesday, March 12, 2025 You can track your package anytime at amazon.com/orders or directly with UPS using your tracking number. Thanks for shopping with us! The Amazon Team", "expected_key_terms": ["Amazon", "1Z879E03YW91234567", "Wednesday, March 12, 2025", "shipped"]}}

{"item": {"email_text": "Good afternoon, Your FedEx package #7749950123456789 from Best Buy is out for delivery today. Our driver will attempt delivery between 10:00 AM - 2:00 PM. Please ensure someone is available to receive your package. If you won't be home, you can redirect your package or schedule a different delivery time at fedex.com. For questions, call 1-800-GO-FEDEX. Best regards, FedEx Customer Service", "expected_key_terms": ["Best Buy", "7749950123456789", "today", "out for delivery"]}}

{"item": {"email_text": "USPS Informed Delivery Daily Digest Your package from VintageFindsShop has been delivered! Tracking #: 9405511206213123456789 Delivered at: 11:47 AM today Status: Left in mailbox Thank you for using USPS. For more information about your deliveries, visit informeddelivery.usps.com. USPS Customer Care", "expected_key_terms": ["**VintageFindsShop**", "**9405511206213123456789**", "**11:47 AM**", "**today**", "**delivered**"]}}

{"item": {"email_text": "DHL Express Delivery Update Dear Valued Customer, We regret to inform you that your shipment from Nike (Waybill: 1234567890) has been delayed due to customs processing requirements. Original delivery date: March 13, 2025 Revised delivery date: Friday, March 14, 2025 Current status: In transit - customs delay We sincerely apologize for this inconvenience and appreciate your patience. DHL Express Team", "expected_key_terms": ["**Nike**", "**1234567890**", "**Friday**", "**March 14, 2025**", "**delayed**", "**in transit**"]}}

{"item": {"email_text": "UPS My Choice Alert Hi there! We have an update on your Target order. Package details: Tracking number: 1Z12345E6605272234 From: Target Status: Order processed - preparing for shipment Estimated ship date: Tomorrow Expected delivery: Monday, March 17, 2025 You'll receive another notification once your package is picked up by UPS. Manage your deliveries at ups.com/mychoice. UPS My Choice", "expected_key_terms": ["Target", "**1Z12345E6605272234**", "**Monday**", "**March 17, 2025**", "**preparing for shipment**"]}}

{"item": {"email_text": "Package Delivery Confirmation Your Apple order has arrived! FedEx has successfully delivered your package today at 3:24 PM. Tracking number: 3712345678901 Delivery location: Front door (signature not required) Status: Delivered Thank you for choosing FedEx for your delivery needs. If you have any concerns about your delivery, please contact us at fedex.com. FedEx Ground", "expected_key_terms": ["**Apple**", "**3712345678901**", "**03:24 PM**", "**delivered**"]}}

{"item": {"email_text": "Priority Mail In Transit Update Your eBay purchase from TechDealsNow is on its way to you. USPS Tracking #: 9405509206092134567890 Current location: Chicago, IL Distribution Center Expected delivery: Saturday, March 15, 2025 by end of business day Status: In transit You'll receive another update when your package is out for delivery. Track your package 24/7 at usps.com. United States Postal Service", "expected_key_terms": ["**TechDealsNow**", "**9405509206092134567890**", "**Saturday**", "**March 15, 2025**", "**in transit**"]}}
```
This bank of example prompts is simple examples of text content for shipping or delivery updates. Included in the `expected_key_terms` are key terms in the format specified in the prompt. This evaluation would check that the contents are included and that the bold formatting is applied to key terms. To check the tone and writing quality, a batch of responses would be better evaluated using an A/B annotated evaluation or other manual human evaluation to ensure quality and produce additional training data.

