# Google Gen AI & Agent Platform — Certification Study Notes


## 1. Prompt Engineering Best Practices ("Unfancy" Prompts)

**Prompt engineering** is the art of writing clear instructions so an AI model gives you the exact answer you want.

Using **"unfancy" prompts** means keeping your wording simple and removing extra fluff. Less noise in your prompt means less chance the model will misunderstand you.

### Core Guidelines

| Best Practice | Why It Matters | Bad Example | Good Example |
| :--- | :--- | :--- | :--- |
| **Be concise** | Shorter prompts reduce noise and cost fewer tokens. | "Could you please maybe tell me a little bit about..." | "Summarize this article in 3 bullet points." |
| **Be specific and well-defined** | Vague prompts produce vague, generic answers. | "Tell me about Earth." | "Generate a list of ways that make Earth unique compared to other planets." |
| **Ask one task at a time** | Mixing unrelated questions confuses the model and lowers quality. | "What's the best method of boiling water and why is the sky blue?" | "What's the best method of boiling water?" |
| **Use system instructions** | Acts as a guardrail to keep the model focused on its role and ignore off-topic requests. | Putting role rules mixed inside the user message. | Setting `system_instruction` in `GenerateContentConfig`. |
| **Turn generative tasks into classification tasks** | Reduces random, unpredictable answers and improves safety by forcing the model to pick from fixed options. | "I'm a high school student. Recommend a programming activity." | "I'm a high school student. Which of these do you suggest and why: a) Python, b) JavaScript, c) Fortran?" |
| **Include examples (shots)** | Shows the model the exact format and tone you expect. | Asking for sentiment without showing an example format. | Showing 1–5 `Tweet -> Sentiment` examples before the real input. |

---

## 2. Prompting Techniques: Zero-Shot, One-Shot, and Few-Shot

"Shots" simply mean **examples** you include inside your prompt. The model learns from these examples on the fly (**in-context learning**).

- **Zero-shot (0 examples):** Best for open-ended, creative tasks.
- **One-shot (1 example):** Shows the exact output format you want.
- **Few-shot (2–5 examples):** Best for consistent, predictable classification or formatting.
  - *Exam Tip:* **1 to 5 examples** is the sweet spot. Too many examples can cause the model to overfit (copy patterns too rigidly) and hurt response quality.

### Code Example: Zero-Shot vs. One-Shot vs. Few-Shot

```python
# Zero-shot: No examples, just instructions
zero_shot_prompt = """Decide whether a Tweet's sentiment is positive, neutral, or negative.

Tweet: I loved the new YouTube video you made!
Sentiment:
"""

# One-shot: 1 example to show the format
one_shot_prompt = """Decide whether a Tweet's sentiment is positive, neutral, or negative.

Tweet: I loved the new YouTube video you made!
Sentiment: positive

Tweet: That was awful. Super boring 😠
Sentiment:
"""

# Few-shot: 2-5 examples for consistent results
few_shot_prompt = """Decide whether a Tweet's sentiment is positive, neutral, or negative.

Tweet: I loved the new YouTube video you made!
Sentiment: positive

Tweet: That was awful. Super boring 😠
Sentiment: negative

Tweet: Something surprised me about this video - it was actually original. Watch it!
Sentiment:
"""
```

---

## 3. Google Gen AI SDK Setup & API Services

Google provides Gemini models through two main API services, and the **Google Gen AI SDK (`google-genai`)** gives you one unified Python interface for both:

| Service | Best For | Authentication / Client Setup |
| :--- | :--- | :--- |
| **Google AI for Developers** | Quick experiments, prototypes, and small projects. | Uses an **API Key** (`genai.Client(api_key="...")`). |
| **Gemini Enterprise Agent Platform** | Production, enterprise-ready projects on Google Cloud. | Uses Google Cloud **Project + Location** with `vertexai=True` (`genai.Client(vertexai=True, project="...", location="...")`). |

> **Certification Tip:** If you omit `vertexai=True` when creating `genai.Client()`, the SDK defaults to the **Google AI for Developers** API, not Google Cloud's Agent Platform.

### Installation & Client Initialization

```python
# Install the official Google Gen AI SDK
# pip install --upgrade --quiet google-genai

from google import genai
from google.genai import types

# Option A: Connect to Agent Platform (Google Cloud)
client = genai.Client(
    vertexai=True,
    project="your-gcp-project-id",
    location="us-central1",
)

# Option B: Connect to Google AI for Developers (prototyping)
# client = genai.Client(api_key="YOUR_GEMINI_API_KEY")

MODEL_ID = "gemini-2.5-flash"  # Replace with your target Gemini model ID
```

---

## 4. Generating Text & Multimodal Content

Gemini is **multimodal**, meaning it can understand **text, images, PDFs, audio, and video** in the same prompt and return text or code.

### Text Generation

```python
response = client.models.generate_content(
    model=MODEL_ID,
    contents="What's the largest planet in our solar system?",
)
print(response.text)
```

### Multimodal Input (Images & Files)

You can pass images in two ways:
1. **Using `PIL.Image`:** Best when you already have an image loaded in Python memory or from a local file.
2. **Using `types.Part.from_uri`:** Best for files stored in **Google Cloud Storage (`gs://...`)** or public URLs without downloading them into memory first.

```python
import requests
from PIL import Image
from google.genai import types

# Method 1: Pass a PIL Image object
image = Image.open(
    requests.get(
        "https://storage.googleapis.com/cloud-samples-data/generative-ai/image/meal.png",
        stream=True,
    ).raw
)

response = client.models.generate_content(
    model=MODEL_ID,
    contents=[
        image,
        "Write a short and engaging blog post based on this picture.",
    ],
)
print(response.text)

# Method 2: Pass a file URI directly (e.g., GCS gs:// bucket or URL)
response = client.models.generate_content(
    model=MODEL_ID,
    contents=[
        types.Part.from_uri(
            file_uri="https://storage.googleapis.com/cloud-samples-data/generative-ai/image/meal.png",
            mime_type="image/png",
        ),
        "Write a short and engaging blog post based on this picture.",
    ],
)
print(response.text)
```

*(In Jupyter notebooks, you can render formatted output cleanly with `from IPython.display import Markdown, display` and `display(Markdown(response.text))`.)*

---

## 5. System Instructions & Model Parameters

In the `google-genai` SDK, settings like system instructions, temperature, safety filters, and schemas all live inside `types.GenerateContentConfig`.

> **SDK Note:** Unlike older Vertex AI SDKs, `system_instruction` is passed **per request** inside `GenerateContentConfig` (or when creating a chat session), not bound once to a model object.

### Key Generation Parameters Explained

| Parameter | What It Does (Simple English) | Typical Range / Tip |
| :--- | :--- | :--- |
| `system_instruction` | Sets the model's permanent persona, rules, and boundaries for the request. | Use this to keep the model on-task. |
| `temperature` | Controls creativity vs. predictability. `0` = deterministic/factual; higher = more creative/random. | Use `0` for code, data extraction, or tool calling; `0.4–1.0` for creative writing. |
| `top_p` | **Nucleus sampling.** Picks from the smallest set of top words whose combined probability adds up to `top_p`. | Default is usually `0.95`. Lower it for more focused text. |
| `top_k` | Limits the model to choosing from only the top `k` most likely next words. | E.g., `20` or `40`. |
| `candidate_count` | Number of different response versions to generate for one prompt. | Usually `1`. |
| `max_output_tokens` | Maximum length of the model's reply (in tokens). | Set this to prevent overly long or expensive outputs. |
| `seed` | A fixed number to make outputs reproducible (same prompt + same seed = very similar output). | Useful for testing and debugging. |
| `stop_sequences` | A list of strings (like `["STOP!"]`) that tell the model to immediately stop generating if it hits them. | Great for controlling formatting. |
| `presence_penalty` | Penalizes words that have **already appeared** at least once, encouraging new topics. | `0.0` = off; positive values = more variety. |
| `frequency_penalty` | Penalizes words **proportional to how often** they appear, reducing repetition. | `0.0` = off; positive values = less word repetition. |

### Code Example: System Instructions & Parameters

```python
from google.genai import types

system_instruction = """
You are a helpful language translator.
Your mission is to translate text in English to French.
"""

response = client.models.generate_content(
    model=MODEL_ID,
    contents="User input: I like bagels.\nAnswer:",
    config=types.GenerateContentConfig(
        system_instruction=system_instruction,
        temperature=0.4,
        top_p=0.95,
        top_k=20,
        candidate_count=1,
        seed=5,
        stop_sequences=["STOP!"],
        presence_penalty=0.0,
        frequency_penalty=0.0,
    ),
)
print(response.text)
```

---

## 6. Safety Filters & Guardrails

The Gemini API includes built-in safety filters so you can block harmful content based on your app's needs.

### Harm Categories & Block Thresholds

| Harm Category (`HarmCategory`) | Block Thresholds (`HarmBlockThreshold` — Strictest to Most Permissive) |
| :--- | :--- |
| `HARM_CATEGORY_HATE_SPEECH` | 1. `BLOCK_LOW_AND_ABOVE` (Strictest — blocks almost anything risky) |
| `HARM_CATEGORY_HARASSMENT` | 2. `BLOCK_MEDIUM_AND_ABOVE` (Balanced default) |
| `HARM_CATEGORY_SEXUALLY_EXPLICIT` | 3. `BLOCK_ONLY_HIGH` (Permissive — blocks only clear violations) |
| `HARM_CATEGORY_DANGEROUS_CONTENT` | 4. `BLOCK_NONE` (Turns blocking off for that category) |

### Code Example: Applying Safety Settings

```python
from google.genai import types

safety_settings = [
    types.SafetySetting(
        category=types.HarmCategory.HARM_CATEGORY_DANGEROUS_CONTENT,
        threshold=types.HarmBlockThreshold.BLOCK_LOW_AND_ABOVE,
    ),
    types.SafetySetting(
        category=types.HarmCategory.HARM_CATEGORY_HARASSMENT,
        threshold=types.HarmBlockThreshold.BLOCK_LOW_AND_ABOVE,
    ),
    types.SafetySetting(
        category=types.HarmCategory.HARM_CATEGORY_HATE_SPEECH,
        threshold=types.HarmBlockThreshold.BLOCK_LOW_AND_ABOVE,
    ),
    types.SafetySetting(
        category=types.HarmCategory.HARM_CATEGORY_SEXUALLY_EXPLICIT,
        threshold=types.HarmBlockThreshold.BLOCK_LOW_AND_ABOVE,
    ),
]

response = client.models.generate_content(
    model=MODEL_ID,
    contents="Write a list of 2 funny things I might say after stubbing my toe in the dark.",
    config=types.GenerateContentConfig(
        safety_settings=safety_settings,
    ),
)
print(response.text)
```

---

## 7. Controlled Generation (Structured JSON Output)

By default, LLMs return free-form text. **Controlled generation** forces the model to reply in **valid JSON** that strictly follows a schema you define. This is essential for connecting AI outputs to databases, APIs, or UI components.

You can define schemas in two ways:
1. **Pydantic Models** (cleanest for Python developers)
2. **Dictionary (JSON Schema subset)** — only these fields are supported: `enum`, `items`, `maxItems`, `nullable`, `properties`, `required`.

### Method 1: Using a Pydantic Model

```python
from pydantic import BaseModel
from google.genai import types

class Recipe(BaseModel):
    name: str
    description: str
    ingredients: list[str]

# Use list[Recipe] when asking for multiple recipes!
response = client.models.generate_content(
    model=MODEL_ID,
    contents="List a few popular cookie recipes and their ingredients.",
    config=types.GenerateContentConfig(
        response_mime_type="application/json",
        response_schema=list[Recipe],
    ),
)
print(response.text)
```

### Method 2: Using a Python Dictionary Schema

*(Note: Fixed the nested array typo from your original notes—for a list of review objects, use a single outer `ARRAY` whose `items` is an `OBJECT`.)*

```python
from google.genai import types

response_schema = {
    "type": "ARRAY",
    "items": {
        "type": "OBJECT",
        "properties": {
            "rating": {"type": "INTEGER"},
            "flavor": {"type": "STRING"},
            "sentiment": {
                "type": "STRING",
                "enum": ["POSITIVE", "NEGATIVE", "NEUTRAL"],
            },
            "explanation": {"type": "STRING"},
        },
        "required": ["rating", "flavor", "sentiment", "explanation"],
    },
}

prompt = """
Analyze the following product reviews, output the sentiment classification and give an explanation.

- "Absolutely loved it! Best ice cream I've ever had." Rating: 4, Flavor: Strawberry Cheesecake
- "Quite good, but a bit too sweet for my taste." Rating: 1, Flavor: Mango Tango
"""

response = client.models.generate_content(
    model=MODEL_ID,
    contents=prompt,
    config=types.GenerateContentConfig(
        response_mime_type="application/json",
        response_schema=response_schema,
    ),
)
print(response.text)
```

---

## 8. Streaming, Async Requests, and Token Counting

### Streaming (`generate_content_stream`)

Normally, `generate_content()` waits until the entire answer is finished before returning anything. **Streaming** sends chunks of text as soon as they are generated, making chatbots feel much faster to users.

```python
for chunk in client.models.generate_content_stream(
    model=MODEL_ID,
    contents="Tell me a short story about a lonely robot who finds friendship.",
):
    print(chunk.text, end="")
```

### Asynchronous Requests (`client.aio`)

If you are building a web backend (like FastAPI) that handles many users at once, use `client.aio` with `await` so your app doesn't freeze while waiting for Gemini to respond.

```python
response = await client.aio.models.generate_content(
    model=MODEL_ID,
    contents="Compose a song about a time-traveling squirrel.",
)
print(response.text)
```

### Counting vs. Computing Tokens

All text, images, and files are broken into **tokens** before the model processes them. Checking token counts helps you control costs and stay within the model's context limit.

| Method | What It Returns | When to Use It |
| :--- | :--- | :--- |
| `client.models.count_tokens(...)` | Only the **total number of input tokens** (`total_tokens`). | Quick pre-flight check for cost and context-window limits (works on both APIs). |
| `client.models.compute_tokens(...)` | Detailed breakdown of the **actual token strings and token IDs** (Agent Platform only). | Debugging how text is split into tokens. |

```python
# Check total token count
count_resp = client.models.count_tokens(
    model=MODEL_ID,
    contents="What's the longest word in the English language?",
)
print("Total tokens:", count_resp.total_tokens)

# Inspect exact tokens and token IDs (Agent Platform)
compute_resp = client.models.compute_tokens(
    model=MODEL_ID,
    contents="What's the longest word in the English language?",
)
print(compute_resp)
```

---

## 9. Function Calling (Tools)

**Function calling** lets Gemini connect to external APIs, databases, or your own Python code.
1. You describe a function's name, purpose, and parameters using `FunctionDeclaration`.
2. Gemini **does not run the code itself**—instead, it replies with the **function name and arguments** it wants you to run (`response.function_calls[0]`).
3. Your app runs the real function and sends the result back to Gemini so it can write the final answer.

> **Best Practice:** Set `temperature=0` when using function calling so the model picks functions and extracts parameters reliably.

```python
from google.genai import types

get_destination = types.FunctionDeclaration(
    name="get_destination",
    description="Get the destination that the user wants to go to",
    parameters={
        "type": "OBJECT",
        "properties": {
            "destination": {
                "type": "STRING",
                "description": "Destination that the user wants to go to",
            },
        },
        "required": ["destination"],
    },
)

destination_tool = types.Tool(function_declarations=[get_destination])

response = client.models.generate_content(
    model=MODEL_ID,
    contents="I'd like to travel to Paris.",
    config=types.GenerateContentConfig(
        tools=[destination_tool],
        temperature=0,
    ),
)

# Inspect the function call suggested by the model
print(response.function_calls[0])
```

*(Bonus SDK Feature: In the `google-genai` SDK, you can also pass a regular Python function directly into `tools=[my_python_func]`, and the SDK can automatically execute it for you!)*

---

## 10. Context Caching

If you send the **same large context** (like a 100-page PDF, a huge codebase, or long system instructions) over and over across many questions, **Context Caching** stores those input tokens in a dedicated cache on Google Cloud.

- **Why use it?** You pay less and get faster responses because you don't re-upload and re-process the same large files every time.
- **Lifecycle:** Create a cache with a **TTL (Time-To-Live)**, reference `cached_content.name` in your requests, and delete the cache when finished to stop storage billing.

```python
from google.genai import types

system_instruction = """
You are an expert researcher. Always stick strictly to the facts in the sources provided.
Answer questions in 1-2 sentences.
"""

pdf_parts = [
    types.Part.from_uri(
        file_uri="gs://cloud-samples-data/generative-ai/pdf/2312.11805v3.pdf",
        mime_type="application/pdf",
    ),
    types.Part.from_uri(
        file_uri="gs://cloud-samples-data/generative-ai/pdf/2403.05530.pdf",
        mime_type="application/pdf",
    ),
]

# 1. Create the cache (expires in 1 hour = 3600s)
cached_content = client.caches.create(
    model=MODEL_ID,
    config=types.CreateCachedContentConfig(
        system_instruction=system_instruction,
        contents=pdf_parts,
        ttl="3600s",
    ),
)

# 2. Query using the cached content
response = client.models.generate_content(
    model=MODEL_ID,
    contents="What is the research goal shared by these research papers?",
    config=types.GenerateContentConfig(
        cached_content=cached_content.name,
    ),
)
print(response.text)

# 3. Clean up the cache when done
client.caches.delete(name=cached_content.name)
```

---

## 11. Text Embeddings

**Text embeddings** turn words or sentences into a **list of numbers (a vector)** that captures their *meaning*. Texts with similar meanings (like *"renew my license"* and *"extend my driver's permit"*) get similar vectors—making embeddings the backbone of **semantic search** and **RAG (Retrieval-Augmented Generation)**.

- **Output Dimensionality:** Embedding models typically output **768 dimensions** by default, but you can shrink `output_dimensionality` (e.g., to `128` or `256`) to save vector database storage and speed up search with minimal accuracy loss.

```python
from google.genai import types

TEXT_EMBEDDING_MODEL_ID = "gemini-embedding-001"  # Check your region's active embedding model

response = client.models.embed_content(
    model=TEXT_EMBEDDING_MODEL_ID,
    contents=[
        "How do I get a driver's license/learner's permit?",
        "How do I renew my driver's license?",
        "How do I change my address on my driver's license?",
    ],
    config=types.EmbedContentConfig(output_dimensionality=128),
)

for embedding in response.embeddings:
    print(f"Vector length: {len(embedding.values)}, first 5 values: {embedding.values[:5]}")
```

