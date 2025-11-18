# LLM Integration - Deep Dive

Multi-provider LLM integration architecture with code references.

## Overview

Letta supports 10+ LLM providers through a unified client interface, enabling easy provider switching.

**Primary Files:**
- [`letta/llm_api/llm_client.py`](../letta-repo/letta/llm_api/llm_client.py) - Unified client factory
- [`letta/llm_api/openai_client.py`](../letta-repo/letta/llm_api/openai_client.py) - OpenAI implementation
- [`letta/llm_api/anthropic_client.py`](../letta-repo/letta/llm_api/anthropic_client.py) - Claude implementation
- [`letta/llm_api/`](../letta-repo/letta/llm_api/) - 10+ provider implementations

---

## Supported Providers

### Cloud Providers

1. **OpenAI** - GPT-4, GPT-4-turbo, GPT-3.5
2. **Anthropic** - Claude 3.5 Sonnet, Claude 3 Opus/Haiku
3. **Google Vertex AI** - Gemini models
4. **Google AI** - Gemini via AI Studio
5. **Azure OpenAI** - Azure-hosted OpenAI models
6. **Groq** - Fast LLaMA inference
7. **Together.ai** - Open-source model hosting
8. **Mistral** - Mistral models
9. **XAI** - Grok models
10. **DeepSeek** - DeepSeek models

### Local Providers

1. **Ollama** - Local LLM runtime
2. **LM Studio** - Local model server
3. **vLLM** - High-performance inference
4. **KoboldCpp** - GGUF model runtime

---

## LLM Client Architecture

### Factory Pattern

**Location:** [`letta/llm_api/llm_client.py`](../letta-repo/letta/llm_api/llm_client.py)

```python
class LLMClient:
    """Factory for creating provider-specific clients."""

    @staticmethod
    def create(
        provider_type: str,                  # "openai", "anthropic", etc.
        put_inner_thoughts_first: bool = True,
        actor: User = None,
    ) -> LLMClientBase:
        """
        Create appropriate LLM client based on provider type.

        Returns provider-specific client implementing LLMClientBase interface.
        """

        if provider_type == "openai":
            return OpenAIClient(...)
        elif provider_type == "anthropic":
            return AnthropicClient(...)
        elif provider_type == "google_vertex":
            return GoogleVertexClient(...)
        elif provider_type == "ollama":
            return OllamaClient(...)
        # ... more providers
        else:
            raise ValueError(f"Unknown provider: {provider_type}")
```

### Base Client Interface

**Location:** [`letta/llm_api/llm_client_base.py`](../letta-repo/letta/llm_api/llm_client_base.py)

```python
class LLMClientBase(ABC):
    """Abstract base class for all LLM clients."""

    @abstractmethod
    async def create_chat_completion_async(
        self,
        messages: List[Dict],
        tools: Optional[List[Dict]] = None,
        tool_choice: Optional[str] = None,
        temperature: float = 0.7,
        max_tokens: int = 1000,
        stream: bool = False,
    ) -> ChatCompletionResponse:
        """
        Unified interface for chat completion.

        All providers must implement this method.
        """
        raise NotImplementedError
```

---

## Provider Implementations

### OpenAI Client

**Location:** [`letta/llm_api/openai_client.py`](../letta-repo/letta/llm_api/openai_client.py)

```python
class OpenAIClient(LLMClientBase):
    async def create_chat_completion_async(
        self,
        messages: List[Dict],
        tools: Optional[List[Dict]] = None,
        **kwargs
    ) -> ChatCompletionResponse:
        """OpenAI API implementation."""

        # Convert to OpenAI format
        openai_messages = self._convert_messages(messages)
        openai_tools = self._convert_tools(tools)

        # Call OpenAI API
        from openai import AsyncOpenAI

        client = AsyncOpenAI(
            api_key=self.api_key,
            base_url=self.base_url,
        )

        response = await client.chat.completions.create(
            model=self.model,
            messages=openai_messages,
            tools=openai_tools,
            **kwargs
        )

        # Convert to unified format
        return self._convert_response(response)
```

### Anthropic Client

**Location:** [`letta/llm_api/anthropic_client.py`](../letta-repo/letta/llm_api/anthropic_client.py)

```python
class AnthropicClient(LLMClientBase):
    async def create_chat_completion_async(
        self,
        messages: List[Dict],
        tools: Optional[List[Dict]] = None,
        **kwargs
    ) -> ChatCompletionResponse:
        """Anthropic (Claude) API implementation."""

        # Anthropic uses different message format
        # - system message separate from messages array
        # - "thinking" blocks for extended thinking

        import anthropic

        client = anthropic.AsyncAnthropic(api_key=self.api_key)

        # Extract system message
        system_message = messages[0]["content"] if messages[0]["role"] == "system" else ""
        user_messages = messages[1:] if system_message else messages

        response = await client.messages.create(
            model=self.model,
            system=system_message,
            messages=user_messages,
            tools=self._convert_tools_to_anthropic(tools),
            **kwargs
        )

        return self._convert_response(response)
```

---

## Configuration

### LLMConfig Schema

**Location:** [`letta/schemas/llm_config.py`](../letta-repo/letta/schemas/llm_config.py)

```python
class LLMConfig(BaseModel):
    """LLM configuration."""

    model: str                               # e.g., "gpt-4", "claude-3-5-sonnet"
    model_endpoint_type: str                 # "openai", "anthropic", etc.
    model_endpoint: Optional[str]            # API endpoint URL
    context_window: int                      # Token limit
    temperature: float = 0.7
    max_tokens: Optional[int] = None

    # Provider-specific
    api_key: Optional[str] = None
    provider_category: Optional[str] = None  # "openai", "anthropic", etc.
```

### Environment Variables

```bash
# OpenAI
OPENAI_API_KEY=sk-...
OPENAI_BASE_URL=https://api.openai.com/v1  # Optional

# Anthropic
ANTHROPIC_API_KEY=sk-ant-...

# Google Vertex AI
GOOGLE_CLOUD_PROJECT=my-project
GOOGLE_APPLICATION_CREDENTIALS=/path/to/credentials.json

# Ollama (local)
OLLAMA_BASE_URL=http://localhost:11434

# Groq
GROQ_API_KEY=...
```

---

## Streaming Support

### Token-by-Token Streaming

**Location:** [`letta/llm_api/openai_client.py`](../letta-repo/letta/llm_api/openai_client.py)

```python
async def create_chat_completion_stream_async(
    self,
    messages: List[Dict],
    **kwargs
) -> AsyncGenerator[ChatCompletionChunk, None]:
    """
    Stream tokens as they're generated.

    Yields ChatCompletionChunk objects.
    """

    client = AsyncOpenAI(...)

    stream = await client.chat.completions.create(
        model=self.model,
        messages=messages,
        stream=True,                         # Enable streaming
        **kwargs
    )

    async for chunk in stream:
        yield self._convert_chunk(chunk)
```

---

## Error Handling

### Retry Logic

```python
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type,
)

@retry(
    retry=retry_if_exception_type(RateLimitError),
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10),
)
async def _call_api_with_retry(self, **kwargs):
    """
    Call API with exponential backoff retry.

    Retries on:
    - RateLimitError (429)
    - APIConnectionError
    """
    return await self.client.chat.completions.create(**kwargs)
```

---

## Provider-Specific Features

### Extended Thinking (Anthropic)

Claude models support extended thinking blocks.

```python
# Response with thinking
{
  "content": [
    {
      "type": "thinking",
      "thinking": "Let me analyze this carefully..."
    },
    {
      "type": "text",
      "text": "Based on my analysis, ..."
    }
  ]
}
```

### Structured Outputs (OpenAI)

GPT-4 supports structured output mode.

```python
response = await client.chat.completions.create(
    model="gpt-4o",
    messages=[...],
    response_format={"type": "json_object"},  # Force JSON
)
```

---

## See Also

- [Agent Execution](./agent-execution.md) - How LLM calls fit in execution loop
- [Streaming System](./streaming-system.md) - Streaming implementation details
