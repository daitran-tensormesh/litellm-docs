import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Tensormesh

## Overview

| Property | Details |
|-------|-------|
| Description | Tensormesh provides serverless and on-demand AI inference with OpenAI-compatible APIs. |
| Provider Route on LiteLLM | `tensormesh/` |
| Link to Provider Doc | [Tensormesh Documentation](https://docs.tensormesh.ai) |
| Default Base URL | `https://serverless.tensormesh.ai/v1` |
| Supported Operations | `/chat/completions`, `/completions`, `/responses`, `/messages` through LiteLLM's Anthropic Messages adapter |

## API Key

```python showLineNumbers title="Environment Variables"
import os

os.environ["TENSORMESH_INFERENCE_API_KEY"] = "your-api-key"
```

## Usage - LiteLLM Python SDK

### Chat Completions

```python showLineNumbers title="Tensormesh Chat Completion"
import os
from litellm import completion

os.environ["TENSORMESH_INFERENCE_API_KEY"] = "your-api-key"

response = completion(
    model="tensormesh/<your-model-name>",
    messages=[{"role": "user", "content": "Say hello in one sentence."}],
)

print(response.choices[0].message.content)
```

### Streaming

```python showLineNumbers title="Tensormesh Streaming Chat Completion"
import os
from litellm import completion

os.environ["TENSORMESH_INFERENCE_API_KEY"] = "your-api-key"

response = completion(
    model="tensormesh/<your-model-name>",
    messages=[{"role": "user", "content": "Write a short poem about inference."}],
    stream=True,
)

for chunk in response:
    print(chunk)
```

### Text Completions

```python showLineNumbers title="Tensormesh Text Completion"
import os
from litellm import text_completion

os.environ["TENSORMESH_INFERENCE_API_KEY"] = "your-api-key"

response = text_completion(
    model="tensormesh/<your-model-name>",
    prompt="Complete this sentence: Fast inference matters because",
    max_tokens=32,
)

print(response.choices[0].text)
```

### Responses API

```python showLineNumbers title="Tensormesh Responses API"
import os
import litellm

os.environ["TENSORMESH_INFERENCE_API_KEY"] = "your-api-key"

response = litellm.responses(
    model="tensormesh/<your-model-name>",
    input="Say hello in one sentence.",
)

print(response)
```

## On-Demand Inference

For on-demand inference, keep the `tensormesh/` provider route and pass both `api_base` and `extra_headers` with `X-User-Id`.

```python showLineNumbers title="On-Demand Environment Variables"
import os

os.environ["TENSORMESH_ON_DEMAND_BASE_URL"] = "https://your-on-demand-route.example.com/v1"
os.environ["TENSORMESH_ON_DEMAND_USER_ID"] = "00000000-0000-0000-0000-000000000000"
```

```python showLineNumbers title="Tensormesh On-Demand Chat Completion"
import os
from litellm import completion

os.environ["TENSORMESH_INFERENCE_API_KEY"] = "your-api-key"

response = completion(
    model="tensormesh/<your-served-model-name>",
    messages=[{"role": "user", "content": "Say hello in one sentence."}],
    api_base=os.environ["TENSORMESH_ON_DEMAND_BASE_URL"],
    extra_headers={"X-User-Id": os.environ["TENSORMESH_ON_DEMAND_USER_ID"]},
)

print(response.choices[0].message.content)
```

The same pattern works for the Responses API:

```python showLineNumbers title="Tensormesh On-Demand Responses API"
import os
import litellm

os.environ["TENSORMESH_INFERENCE_API_KEY"] = "your-api-key"

response = litellm.responses(
    model="tensormesh/<your-served-model-name>",
    input="Say hello in one sentence.",
    api_base=os.environ["TENSORMESH_ON_DEMAND_BASE_URL"],
    extra_headers={"X-User-Id": os.environ["TENSORMESH_ON_DEMAND_USER_ID"]},
)

print(response)
```

## Usage - LiteLLM Proxy

Add Tensormesh to your LiteLLM Proxy configuration:

```yaml showLineNumbers title="config.yaml"
model_list:
  - model_name: tensormesh-chat
    litellm_params:
      model: tensormesh/<your-model-name>
      api_key: os.environ/TENSORMESH_INFERENCE_API_KEY

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
```

For on-demand inference, set `api_base` to your on-demand route and pass `X-User-Id` in `extra_headers`:

```yaml showLineNumbers title="config.yaml"
model_list:
  - model_name: tensormesh-on-demand
    litellm_params:
      model: tensormesh/<your-served-model-name>
      api_key: os.environ/TENSORMESH_INFERENCE_API_KEY
      api_base: os.environ/TENSORMESH_ON_DEMAND_BASE_URL
      extra_headers:
        X-User-Id: os.environ/TENSORMESH_ON_DEMAND_USER_ID

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
```

Start the proxy:

```bash showLineNumbers title="Start LiteLLM Proxy"
export TENSORMESH_INFERENCE_API_KEY="your-api-key"
export LITELLM_MASTER_KEY="sk-local-tensormesh"
litellm --config config.yaml --port 4000

# RUNNING on http://0.0.0.0:4000
```

Requests to LiteLLM Proxy must use the proxy key in `Authorization: Bearer $LITELLM_MASTER_KEY`. `TENSORMESH_INFERENCE_API_KEY` is only used by LiteLLM when it calls Tensormesh upstream.

For a basic startup check, use `/health/liveliness` or `/health/readiness`. The `/health` endpoint is authenticated and may run model checks.

<Tabs>
<TabItem value="openai-sdk" label="OpenAI SDK">

```python showLineNumbers title="Tensormesh via Proxy - OpenAI SDK"
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:4000",
    api_key="sk-local-tensormesh",
)

response = client.chat.completions.create(
    model="tensormesh-chat",
    messages=[{"role": "user", "content": "hello from litellm"}],
)

print(response.choices[0].message.content)
```

</TabItem>

<TabItem value="curl" label="cURL">

```bash showLineNumbers title="Tensormesh via Proxy - cURL"
curl http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -d '{
    "model": "tensormesh-chat",
    "messages": [{"role": "user", "content": "hello from litellm"}]
  }'
```

</TabItem>
</Tabs>

## Anthropic Messages Compatibility

LiteLLM can translate Anthropic Messages-shaped requests to Tensormesh chat completions. In the Python SDK, use the Anthropic Messages facade:

```python showLineNumbers title="Anthropic Messages through LiteLLM SDK"
import os
import litellm

os.environ["TENSORMESH_INFERENCE_API_KEY"] = "your-api-key"

response = litellm.anthropic.messages.create(
    model="tensormesh/<your-model-name>",
    max_tokens=128,
    messages=[{"role": "user", "content": "Say hello in one sentence."}],
)

print(response["content"][0]["text"])
```

For HTTP clients, LiteLLM Proxy exposes the Anthropic-compatible `/v1/messages` endpoint and routes the upstream request to Tensormesh chat completions.

For serverless, use the `tensormesh-chat` proxy model from the config above. For on-demand, use the on-demand proxy model alias. In the `/v1/messages` request body, set `model` to the same value as the proxy `model_name`.

```bash showLineNumbers title="Anthropic Messages through LiteLLM Proxy"
curl http://localhost:4000/v1/messages \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "tensormesh-chat",
    "max_tokens": 128,
    "messages": [{"role": "user", "content": "Say hello in one sentence."}]
  }'
```

Both the SDK facade and the Proxy `/v1/messages` endpoint use LiteLLM's Anthropic Messages adapter. Tensormesh receives an OpenAI-compatible chat completion request upstream.

For on-demand usage, use the same proxy pattern with `api_base` and `extra_headers` in the model config.

## Common Parameters

These are the common parameters to start with. Additional parameters are model-dependent and should be validated with the target Tensormesh model.

| Endpoint | Common parameters |
|----------|-------------------|
| `/chat/completions` | `messages`, `max_tokens`, `max_completion_tokens`, `temperature`, `top_p`, `stream`, `stop`, `tools`, `tool_choice`, `response_format`, `extra_headers` |
| `/completions` | `prompt`, `max_tokens`, `temperature`, `top_p`, `stream`, `stop` |
| `/responses` | `input`, `max_output_tokens`, `temperature`, `top_p`, `stream`, `tools`, `tool_choice`, `text`, `extra_headers` |
| `/messages` | `messages`, `max_tokens`, `temperature`, `top_p`, `stream`, `tools`, `tool_choice`, `extra_headers` |

For chat completions, LiteLLM accepts `max_completion_tokens` and maps it to `max_tokens` for Tensormesh.

## Notes

- Use `model="tensormesh/<your-model-name>"` for direct LiteLLM SDK calls.
- The default serverless base URL is `https://serverless.tensormesh.ai/v1`.
- On-demand base URLs should include `/v1`.
