# 03 — LLM Core Concepts, Transformer Architecture & Inference

> **Context**: Theoretical & Infrastructure Depth. Explains Transformer mechanics, KV Cache, quantization, and Low-Rank Adaptation (LoRA).

---

## 1. The Transformer Architecture Demystified

### 1.1 Self-Attention Mechanism
At the heart of the Transformer architecture is **Scaled Dot-Product Attention**:

$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{Q K^T}{\sqrt{d_k}}\right) V$$

```
Input Tokens: ["The", "database", "deadlocked"]
     │
     ▼ Embedding + Positional Encoding (RoPE)
[ X ] Matrix (N tokens × D dimension)
     │
     ├──► W_q ──► Query Matrix (Q) ──┐
     ├──► W_k ──► Key Matrix (K)   ──┴──► (Q · K^T) / sqrt(d_k) ──► Softmax (Attention Weights)
     └──► W_v ──► Value Matrix (V) ──────────────────────────────────────────┴──► Weighted Values
```

1. **Queries ($Q$)**: What each token is looking for.
2. **Keys ($K$)**: What each token contains / offers.
3. **Values ($V$)**: The actual information extracted if a match occurs.
4. **$\frac{1}{\sqrt{d_k}}$ Scaling Factor**: Prevents dot products from exploding into large magnitudes for high dimensions ($d_k$), which would push softmax gradients into vanishingly small regions.

### 1.2 Multi-Head Attention (MHA)
Instead of performing a single attention function, Multi-Head Attention projects $Q, K, V$ into $H$ distinct lower-dimensional subspaces, allowing the model to attend simultaneously to syntactic structure, semantic relationships, and distant antecedent references.

---

## 2. LLM Inference Optimization: KV Cache & PagedAttention

### 2.1 The KV Cache (Key-Value Cache)
During autoregressive text generation, each new token depends on all previous tokens.
- **Without KV Cache**: At token $N$, the model must re-project and re-compute $Q, K, V$ matrices for all tokens $1 \dots N-1$. Inference time scales **quadratically $O(N^2)$**.
- **With KV Cache**: The Key and Value matrices for all previous tokens are stored in GPU VRAM. Generating the next token only requires computing $Q$ for the single new token and multiplying it against the cached $K$ and $V$ tensors. Inference time drops to **linear $O(N)$**.

### 2.2 Memory Footprint of the KV Cache
```
Memory = 2 (K and V) × 2 (bytes in FP16) × n_layers × n_heads × d_head × seq_length × batch_size
```
For a 70B parameter model with sequence length 4,096 at batch size 16:
The KV Cache consumes **~26 GB of GPU VRAM alone**, often exceeding the memory required for the model weights themselves!
- **Solution (PagedAttention / vLLM)**: Partitions the KV cache into non-contiguous virtual memory pages (analogous to OS virtual memory), eliminating memory fragmentation and enabling 2x-4x higher throughput.

---

## 3. Model Quantization: Running 70B Models on Commodity GPUs

Model weights are traditionally trained in **FP32** (32-bit floating point, 4 bytes per parameter) or **FP16 / BF16** (16-bit, 2 bytes per parameter).

$$\text{VRAM Required} \approx \text{Parameters (Billions)} \times \text{Bytes per Parameter} \times 1.2 \text{ (overhead)}$$

| Precision Tier | Bits | Bytes / Parameter | VRAM for 70B Model | Hardware Required |
|---|---|---|---|---|
| **FP16 / BF16** | 16-bit | 2.0 B | **~140 GB** | 2× A100 (80GB) or 4× RTX 4090 |
| **INT8** | 8-bit | 1.0 B | **~75 GB** | 1× A100 (80GB) |
| **INT4 (AWQ / GPTQ)** | 4-bit | 0.5 B | **~38 GB** | 1× A6000 (48GB) or Mac Studio M2 Max |

### Quantization Techniques:
1. **GPTQ / AWQ (Post-Training Quantization)**: Compresses weights to 4-bit integers with minimal perplexity degradation by preserving salient outlier weights.
2. **GGUF (llama.cpp)**: Optimized CPU/Metal format allowing split memory offloading across unified RAM and GPU cores.

---

## 4. Parameter-Efficient Fine-Tuning: LoRA & QLoRA

### 4.1 Low-Rank Adaptation (LoRA)
Full fine-tuning of a 70B model requires updating all 70 billion parameters and maintaining optimizer states (Adam requires 8 bytes per parameter), consuming over **800 GB of VRAM**.

**LoRA Principle**: The weight updates $\Delta W$ during domain adaptation have a very low "intrinsic dimension".
Instead of training full matrix $\Delta W \in \mathbb{R}^{d \times k}$, LoRA decomposes it into two low-rank matrices:

$$\Delta W = B \cdot A, \quad \text{where } B \in \mathbb{R}^{d \times r}, \ A \in \mathbb{R}^{r \times k}, \quad r \ll \min(d, k)$$

```
Original Weight Matrix (W0) ──► Frozen in memory (0 updates)
           │
           ├──► Low-Rank A (d × r) ──► Low-Rank B (r × k) ──► Output = (W0 · x) + (B · A · x)
```
- Rank $r$ is typically set to **$8$ or $16$**.
- Reduces trainable parameters by **over 99.8%**, cutting VRAM requirements from 800GB to under 24GB.

### 4.2 QLoRA (Quantized LoRA)
Quantizes the base frozen model to **4-bit NormalFloat (NF4)** and attaches 16-bit LoRA adapter matrices. Enables fine-tuning a 70B parameter model on a **single 48GB GPU**.

---

## 5. Senior Interview Q&A Cheatsheet

### Q1: "Why does temperature in an LLM affect output determinism, and what happens at temperature = 0?"
> **Answer**: In the final generation layer, the model produces a vector of unnormalized log-probabilities (logits $z_i$). The logits pass through a temperature-scaled softmax function:
> $$P(w_i) = \frac{\exp(z_i / T)}{\sum_j \exp(z_j / T)}$$
> - When **$T \to 0$ (Greedy Decoding)**: The highest logit dominates completely ($P(\text{argmax } z_i) \to 1$), making output selection fully deterministic.
> - When **$T > 1.0$**: Logit differences are flattened, raising the probability of lower-ranked tokens and increasing creative variance / hallucination risk.
> - In production APIs for financial math, code generation, or structured JSON extraction, set **temperature to $0.0$ - $0.2$**.

### Q2: "What is RoPE (Rotary Position Embedding) and why has it replaced sinusoidal position embeddings in modern LLMs?"
> **Answer**:
> Traditional sinusoidal embeddings add absolute position vectors to token embeddings, struggling to generalize to sequences longer than those seen in training.
> **Rotary Position Embedding (RoPE)** encodes positional information by **rotating Query and Key vectors in the complex 2D plane** by an angle proportional to token position. This guarantees that the dot product $Q_m \cdot K_n$ depends purely on the **relative distance $(m - n)$** between tokens. RoPE enables modern models (Llama 3, Mistral) to smoothly extrapolate context windows from 4K to 128K+ tokens via simple frequency base scaling.
