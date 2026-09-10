# LLM Workflow

This repo has two companion pieces, each answering a different question:

- **`index.html`** — *What does a language model actually do to my text?*
  An educational, interactive walkthrough of the model-internal pipeline: tokenization, embeddings, attention, and sampling.

- **This README (below)** — *How does a request flow through a production system serving an LLM?*
  An engineering/ops reference diagram: load balancers, caching, monitoring, and the infrastructure around the model.

---

## System Workflow

From input to output — step by step.
```mermaid
flowchart TD
    User["👤 User / Client<br/><small>Web app / Browser / API / Other clients<br/>(text input)</small>"]
    LB["🔀 Load Balancer<br/><small>(distributes traffic)</small>"]

    subgraph PREP["Model-Serving Layer (Preprocessing)"]
        direction LR
        Gateway["🛡️ API Gateway<br/><small>auth, rate limit, logging</small>"]
        Tokenizer["🔡 Tokenizer<br/><small>text → tokens</small>"]
        Embed["🔢 Embed<br/><small>tokens → vectors</small>"]
        Position["➰ Position<br/><small>add positional encoding</small>"]
        Gateway --> Tokenizer --> Embed --> Position
    end

    subgraph TRANSFORMER["LLM Layers — Transformer Block × N"]
        direction LR
        Attention["👁️ Attention<br/><small>self-attention</small>"]
        FFN["🔀 FFN<br/><small>feed-forward network</small>"]
        Attention --> FFN
        FFN -. "repeat N times" .-> Attention
    end

    Cache[("🗄️ Cache (Redis)<br/><small>• session history<br/>• RAG/retrieval embeddings<br/>• semantic response cache</small>")]

    Sampling["🎯 Output Projection + Sampling<br/><small>final linear layer → softmax → pick next token</small>"]
    PostProc["⚙️ Post-processing<br/><small>detokenize, format, safety checks, response shaping</small>"]
    Response["💬 Response to User<br/><small>text / JSON / stream</small>"]

    subgraph INFRA["Supporting Infrastructure"]
        direction LR
        DB[("🗃️ Database<br/><small>user data, logs, analytics</small>")]
        Monitor["📈 Monitoring & Logging<br/><small>metrics, error logs, performance</small>"]
        External["☁️ External Services<br/><small>model provider, APIs, integrations</small>"]
    end

    User -->|"1. Send request"| LB
    LB -->|"2. Route to service"| PREP
    PREP -->|"3. Processed input"| TRANSFORMER
    TRANSFORMER -->|"4. Hidden states"| Sampling
    Sampling -->|"5. Generate response"| PostProc
    PostProc -->|"6. Send to output"| Response

    TRANSFORMER <-. "check / store" .-> Cache

    PREP -.-> INFRA
    TRANSFORMER -.-> INFRA
    Response -.-> INFRA

    classDef userStyle fill:#f4ddf7,stroke:#1b1b3a,stroke-width:2px
    classDef lbStyle fill:#cfe3fb,stroke:#1b1b3a,stroke-width:2px
    classDef prepStyle fill:#fdf3cf,stroke:#1b1b3a,stroke-width:2px
    classDef transStyle fill:#e3daf9,stroke:#1b1b3a,stroke-width:2px
    classDef cacheStyle fill:#c3ece2,stroke:#1b1b3a,stroke-width:2px
    classDef samplingStyle fill:#fde2d0,stroke:#1b1b3a,stroke-width:2px
    classDef postStyle fill:#fbcfcf,stroke:#1b1b3a,stroke-width:2px
    classDef respStyle fill:#c9f0d8,stroke:#1b1b3a,stroke-width:2px
    classDef infraStyle fill:#eeeeee,stroke:#1b1b3a,stroke-width:1px

    class User userStyle
    class LB lbStyle
    class Gateway,Tokenizer,Embed,Position prepStyle
    class Attention,FFN transStyle
    class Cache cacheStyle
    class Sampling samplingStyle
    class PostProc postStyle
    class Response respStyle
    class DB,Monitor,External infraStyle
```

**Scalable • Secure • Fast**

## Notes on this version

- **Transformer block wraps Attention + FFN together** (`N ×`), rather than showing them as three sequential steps — a real transformer repeats the *pair*, not each part independently.
- **Cache (Redis)** is labeled for what a Redis-style cache is actually used for in production — session history, retrieval/RAG embeddings, semantic response caching — not the GPU-resident KV-cache used during inference itself, which is a different mechanism.
- **Added an "Output Projection + Sampling" step** between the transformer layers and post-processing, since converting final hidden states into an actual next token (linear layer → softmax → sampling) is a distinct step that was previously skipped.
- **Tokenizer / Embed / Position** are grouped as a "Model-Serving Layer," separate from the API Gateway, since they're part of model input preparation rather than generic backend infrastructure.

> This diagram uses [Mermaid](https://mermaid.js.org/), which GitHub renders natively — just keep this file as `README.md` (or embed the code block in any README) and it will display as a diagram automatically.
