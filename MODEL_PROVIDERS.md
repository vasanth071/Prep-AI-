# PrepAI — Model Providers & Hardware Setup Guide

## 1. Overview of Supported Providers

PrepAI's Provider Abstraction Layer enables seamless switching between 100% private on-premise local inference and major commercial cloud AI providers without altering core business logic.

```
                      ┌─────────────────────────────────┐
                      │    AIProviderFactory (:7857)    │
                      └────────────────┬────────────────┘
                                       │
        ┌──────────────┬───────────────┴───────────────┬──────────────┐
        ▼              ▼                               ▼              ▼
┌──────────────┐ ┌──────────────┐             ┌────────────────┐ ┌──────────────┐
│ Local Ollama │ │Anthropic API │             │ Google Gemini  │ │OpenAI / Groq │
│ (Default)    │ │(Claude 3.5)  │             │(Flash / Pro)   │ │(vLLM / LM)   │
└──────────────┘ └──────────────┘             └────────────────┘ └──────────────┘
```

---

## 2. Local Ollama Provider (Primary Default)

Ollama provides private, zero-cost, local inference running entirely within your workstation or dedicated server.

### 2.1. Prerequisites & Installation
1. Install Ollama for Windows, Linux, or macOS from [ollama.com](https://ollama.com).
2. Verify daemon is listening:
   ```bash
   curl http://127.0.0.1:11434/api/tags
   ```

### 2.2. Recommended Model Suite
PrepAI utilizes three distinct models for optimal performance and latency:

```bash
# 1. Primary Reasoning & Technical Evaluation (Mandatory)
ollama pull qwen2.5-coder:7b

# 2. Dense Vector Embeddings (Mandatory)
ollama pull nomic-embed-text

# 3. Fast Conversational Turns / Voice Follow-ups (Optional, Recommended for Low-End CPUs)
ollama pull llama3.2:latest
```

### 2.3. Minimum & Recommended Hardware Specifications

| Component | Minimum Spec (CPU Only) | Recommended Spec (NVIDIA GPU) | Enterprise Spec |
| :--- | :--- | :--- | :--- |
| **Processor** | 8-core modern x86_64 / Apple Silicon | Intel i7 / Ryzen 7 / Apple M1 Pro | Intel Xeon / AMD Threadripper |
| **System RAM**| 16 GB DDR4/DDR5 | 32 GB DDR5 | 64 GB ECC RAM |
| **GPU / VRAM**| Integrated Graphics (CPU inference) | NVIDIA RTX 3060 (12 GB VRAM) | NVIDIA RTX 4090 (24 GB) or A100 |
| **Storage** | 20 GB SSD space | 50 GB NVMe PCIe 4.0 | 100 GB Enterprise NVMe |
| **Eval Speed**| ~8-12 seconds / turn | **~2.5-4.5 seconds / turn** | **~0.8-1.5 seconds / turn** |

---

## 3. Cloud Provider Setup & Configuration

To enable cloud fallbacks or run cloud-first, populate the corresponding environment variables in `server/.env`.

### 3.1. Anthropic Claude
- **Supported Models**: `claude-3-5-haiku-20241022`, `claude-3-7-sonnet-20250219`
- **Key Configuration**:
  ```env
  AI_PROVIDER=anthropic
  ANTHROPIC_API_KEY=sk-ant-api03-...
  ANTHROPIC_MODEL=claude-3-5-haiku-20241022
  ```
- **Strengths**: Best-in-class nuanced reasoning on complex system design trade-offs.

### 3.2. Google Gemini
- **Supported Models**: `gemini-1.5-flash`, `gemini-1.5-pro`, `gemini-2.0-flash`
- **Key Configuration**:
  ```env
  AI_PROVIDER=gemini
  GEMINI_API_KEY=AIzaSy...
  GEMINI_MODEL=gemini-1.5-flash
  ```
- **Strengths**: Highly cost-effective with massive context window handling.

### 3.3. OpenAI & OpenAI-Compatible (Groq, DeepSeek, vLLM, LM Studio)
- **Supported Endpoints**: Any provider implementing the standard `/v1/chat/completions` protocol.
- **Key Configuration**:
  ```env
  AI_PROVIDER=openai
  OPENAI_API_KEY=your_api_key_or_dummy
  OPENAI_BASE_URL=https://api.openai.com/v1   # Or https://api.groq.com/openai/v1 or http://localhost:8000/v1
  OPENAI_MODEL=gpt-4o-mini
  ```
- **Groq Acceleration**: Set `OPENAI_BASE_URL=https://api.groq.com/openai/v1` and `OPENAI_MODEL=llama-3.3-70b-versatile` for ultra-low latency (< 800ms) turns.

---

## 4. Environment Variables Reference

| Variable | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `AI_PROVIDER` | `string` | `ollama` | Active provider: `ollama`, `anthropic`, `gemini`, `openai`. |
| `OLLAMA_BASE_URL` | `url` | `http://127.0.0.1:11434` | Network location of the local Ollama daemon. |
| `OLLAMA_MODEL` | `string` | `qwen2.5-coder:7b` | Primary chat completion and evaluation model. |
| `OLLAMA_EMBED_MODEL`| `string` | `nomic-embed-text` | Embedding model for semantic vector similarity. |
| `ANTHROPIC_API_KEY`| `string` | `""` | Anthropic console authentication token. |
| `ANTHROPIC_MODEL` | `string` | `claude-3-5-haiku` | Anthropic model version identifier. |
| `GEMINI_API_KEY` | `string` | `""` | Google AI Studio authentication API key. |
| `GEMINI_MODEL` | `string` | `gemini-1.5-flash` | Google Gemini model family identifier. |
| `OPENAI_API_KEY` | `string` | `""` | OpenAI or third-party proxy secret key. |
| `OPENAI_BASE_URL` | `url` | `https://api.openai.com/v1` | Base URL for OpenAI-compatible proxies/vLLM. |
| `OPENAI_MODEL` | `string` | `gpt-4o-mini` | Model parameter passed to completion endpoint. |

---

## 5. Failover Matrix & Fallback Order

PrepAI guarantees continuous session operation via structured fallback resolution:

```
┌────────────────────────────────┐
│  Primary: Local Ollama         │
└───────────────┬────────────────┘
                │ (Connection Refused / Timeout > 45s)
                ▼
┌────────────────────────────────┐
│  Fallback 1: Claude / Gemini   │ (If API keys present)
└───────────────┬────────────────┘
                │ (Cloud Quota Exhausted / Network Offline)
                ▼
┌────────────────────────────────┐
│  Fallback 2: OpenAI Compatible │
└───────────────┬────────────────┘
                │ (All Providers Unavailable)
                ▼
┌────────────────────────────────┐
│  Deterministic Safe Engine     │ (Lexical Jaccard + Keyword Cluster Analysis)
└────────────────────────────────┘
```

The system continuously attempts to self-heal back to the primary local provider when health pings succeed.
