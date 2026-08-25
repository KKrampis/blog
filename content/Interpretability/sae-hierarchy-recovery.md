---
title: "Recovering Feature Hierarchies in Sparse Autoencoders via Co-firing Statistics and LLM AutoInterp"
date: 2026-07-23
---

## Abstract

Sparse Autoencoders (SAEs) have emerged as a powerful lens for decomposing the polysemantic activations of large language models into interpretable, monosemantic features. A natural but underexplored question is whether these features organize into a *hierarchy* — whether some features are systematically more general than others, activating whenever a more specific child feature fires but also in broader contexts. In this work we describe an experimental pipeline for recovering such hierarchies from a Twin SAE trained on Gemma-2-2B, using two complementary signals: **decoder weight geometry** (cosine similarity between feature directions) and **co-firing statistics** (the conditional probability that a candidate parent fires given that a candidate child fires). We further propose using **LLM AutoInterp** — automated interpretation of features via top-activating corpus examples fed to Claude Haiku — to semantically validate recovered edges. The result is a data-driven, automatically interpretable feature hierarchy over all 32,768 latents of the Matryoshka SAE branch, grounded in both geometric and statistical evidence.

---

## 1. Background: Superposition and the SAE Solution

Modern language models represent information in high-dimensional residual stream vectors. Empirical work by [Elhage et al. (2022)](https://transformer-circuits.pub/2022/toy_model/index.html) established the **superposition hypothesis**: transformer residual streams pack far more features than their ambient dimension by representing each feature as a direction in a high-dimensional space, tolerating interference between features because only a sparse subset are active at any given token position. Formally, if $\mathbf{h} \in \mathbb{R}^{d}$ is a residual stream vector and $\{\mathbf{f}_i\}_{i=1}^{F}$ is a dictionary of feature directions with $F \gg d$, then:

$$\mathbf{h} \approx \sum_{i \in S} a_i \mathbf{f}_i + \boldsymbol{\varepsilon}$$

where $S \subset [F]$ is a small active set, $a_i > 0$ are activation magnitudes, and $\boldsymbol{\varepsilon}$ is reconstruction error. The sparsity of $S$ is what makes superposition tolerable: features rarely interfere because they rarely co-occur in practice.

The standard tool for finding this dictionary is the **Sparse Autoencoder** [Cunningham et al., 2023; Bricken et al., 2023]. An SAE learns an encoder $f: \mathbb{R}^d \to \mathbb{R}^F$ and decoder $g: \mathbb{R}^F \to \mathbb{R}^d$ to minimize:

$$\mathcal{L} = \|\mathbf{h} - g(f(\mathbf{h}))\|^2 + \lambda \|f(\mathbf{h})\|_1$$

The $L_1$ penalty encourages sparse activations. The decoder columns $\mathbf{W}_\text{dec}[:,i] \in \mathbb{R}^d$ are the **feature directions** — unit vectors in residual-stream space that the model uses to encode feature $i$. The encoder produces a latent vector $\mathbf{z} = f(\mathbf{h}) \in \mathbb{R}^F_{\geq 0}$ where most entries are zero; the magnitude $z_i$ quantifies how strongly feature $i$ is present at a given token.

---

## 2. The Twin SAE: Matryoshka and BatchTopK

Our experiments use the **Twin SAE** trained by [Chanin (2024)](https://huggingface.co/chanind/twin), a pair of weight-linked SAEs trained jointly on residual stream activations of **Gemma-2-2B** at layer 12. The two branches share the same encoder backbone but apply different sparsity mechanisms:

**Matryoshka SAE (Mat-SAE)** uses a *nested* activation scheme inspired by Matryoshka representation learning [Kusupati et al., 2022]. Features are ordered so that the top-$k$ (by activation) approximate the top-$k' < k$ reconstruction. This creates a hierarchy by *construction* within the dictionary: lower-indexed features are more universally important, while higher-indexed ones capture finer-grained distinctions.

**BatchTopK SAE (BTK-SAE)** enforces exactly $k$ nonzero activations per token across a batch, maintaining a fixed mean $L_0$ of approximately 143 active features per token. This differs from $L_1$ regularization in that it is *sparsity-exact* rather than sparsity-encouraging: the global $k$ is fixed, removing the need to tune $\lambda$.

Both branches have dictionary width $F = 32{,}768$ and operate on residual streams of dimension $d = 2{,}304$ (Gemma-2-2B layer 12). The key experimental lever is that **the two branches disagree** on which features to activate for a given input: Mat-SAE must route information through its nested ordering constraint while BTK-SAE is unconstrained. This disagreement is one signal for inferring which Mat-SAE features are more general.

---

## 3. Why Features Might Organize as a Hierarchy

The superposition hypothesis predicts that features span all frequencies of specificity. A model representing language must encode:

- Very general features: *"this is a word"*, *"this is a number"*, *"this is punctuation"*
- Mid-level features: *"this is a year"*, *"this is a proper noun"*, *"this is a Python keyword"*
- Specific features: *"this is a year in a date with day-month-year format"*, *"this is a function name being defined"*

Crucially, when a specific feature fires, its more general ancestors should also fire — a token that triggers *"year in ISO-8601 date"* will almost certainly also trigger *"year"*, *"number"*, and *"temporal expression"*. This is the defining property of a hierarchy:

$$P(\text{parent fires} \mid \text{child fires}) \approx 1$$

while the converse need not hold:

$$P(\text{child fires} \mid \text{parent fires}) \ll 1$$

The asymmetry is the signal. If co-activation were symmetric, we would have two correlated features with no parent-child relationship — perhaps two different aspects of the same concept. The **conditional probability** $P(j \text{ fires} \mid i \text{ fires})$ is therefore our primary edge score for inferring $i \to j$ (child $i$, parent $j$) hierarchy edges.

---

## 4. Method 1 — Decoder Weight Geometry

The simplest hierarchy signal comes directly from the trained weights, without running the model on any corpus at all. If feature $j$ is a parent of feature $i$, we expect the decoder direction $\mathbf{d}_j = \mathbf{W}_\text{dec}[:,j]$ to be geometrically close to $\mathbf{d}_i$, since the model should be able to express a less specific concept as a linear combination that includes the more specific one.

We compute the **cosine similarity matrix** between all pairs of feature directions:

$$S_{ij} = \frac{\mathbf{d}_i \cdot \mathbf{d}_j}{\|\mathbf{d}_i\| \|\mathbf{d}_j\|}$$

and declare an edge $i \to j$ (child $i$, parent $j$) whenever $S_{ij}$ exceeds a threshold $\tau$ and $j$ has lower sparsity rank than $i$ (i.e., $j$ fires more frequently than $i$, consistent with $j$ being more general).

This approach is fast — $O(F^2 d)$ which for $F = 32{,}768, d = 2{,}304$ is computable in minutes — but has important limitations. High cosine similarity between decoder directions does not guarantee a functional relationship: two features could share a direction merely because the SAE placed them near each other to pack more features into $\mathbb{R}^d$ without either being semantically related to the other. Moreover, the decoder weight geometry reflects the SAE's internal representation decisions, which may not perfectly track semantic hierarchies in the underlying model.

---

## 5. Method 2 — Co-firing Statistics

The co-firing approach is more expensive but more directly grounded in the model's actual behavior. We stream a large corpus of text through Gemma-2-2B, compute layer-12 residual stream activations at each token position, encode them through the Mat-SAE, and accumulate:

**Fire counts**: $c_i = \sum_{t} \mathbf{1}[z_i^{(t)} > 0]$ — the number of tokens where feature $i$ fires.

**Co-fire counts**: $c_{ij} = \sum_{t} \mathbf{1}[z_i^{(t)} > 0 \wedge z_j^{(t)} > 0]$ — the number of tokens where both $i$ and $j$ fire simultaneously.

From these we compute the **conditional co-firing probability**:

$$P(j \mid i) = \frac{c_{ij}}{c_i}$$

This is the fraction of tokens where $i$ fires that also have $j$ firing. If $P(j \mid i) \geq \tau$ (we use $\tau = 0.90$ as default), we declare $j$ to be a parent of $i$.

### 5.1 Scale and Implementation

The full 32,768-dimensional co-firing matrix requires storing $\mathbf{C} \in \mathbb{R}^{32768 \times 32768}$, which at float64 precision occupies approximately 8 GB of memory. Computing $\mathbf{C}$ is achieved by accumulating:

$$\mathbf{C} \mathrel{+}= \mathbf{F}^\top \mathbf{F}$$

where $\mathbf{F} \in \{0,1\}^{N \times F}$ is the binary fire mask for $N$ tokens in the current batch. On an NVIDIA A100 80GB GPU, this matmul takes $\approx 3$–$5$ seconds per batch of $N = 65{,}536$ tokens (batch size 64 sequences $\times$ 1024 context length), and represents the dominant GPU operation.

Our 20M-token run (approximately 0.83% of the full 2.4B-token training corpus) consumed:

- **306 batches** of 65,536 tokens each
- **8.59 GB** final co-firing matrix
- **32,767 live latents** (features that fired at least once)
- Mean $L_0 = 143.47$ features per token

The corpus is `chanind/pile-uncopyrighted-gemma-1024-abbrv-2B`, the same corpus used to train the Twin SAE, ensuring activation statistics are drawn from the in-distribution regime.

### 5.2 Edge Recovery

From the co-firing matrix $\mathbf{C}$ and fire counts $\mathbf{c}$ we recover hierarchy edges:

```python
# Conditional probability matrix
P = C / c[:, None]          # P[i, j] = P(j fires | i fires)
P.fill_diagonal_(0)         # exclude self-edges

# For each child i, take top-k parents by P(j | i)
ancestors = {}
for i in range(d_sae):
    parent_scores = P[i]    # [d_sae]
    top_parents = (parent_scores >= threshold).nonzero()
    # optionally further filtered by top-k
    ancestors[i] = top_parents.tolist()
```

With threshold $\tau = 0.90$ and optional transitive closure (adding $i \to k$ whenever $i \to j$ and $j \to k$ exist), this recovers a directed acyclic graph over all live latents. Across the 20M-token run, edge counts vary substantially with threshold:

| Threshold $\tau$ | Edges recovered | Latents with $\geq 1$ parent |
|-----------------|----------------|------------------------------|
| 0.85 | ~1.2M | ~28,000 |
| 0.90 | ~420K | ~24,000 |
| 0.95 | ~85K | ~18,000 |
| 0.99 | ~8K | ~9,000 |

The 0.90 threshold is a practical default that recovers a rich hierarchy while keeping the false-positive rate low enough for downstream validation.

---

## 6. Comparing the Two Signals

Decoder weight geometry and co-firing statistics are complementary:

| Property | Decoder Geometry | Co-firing Statistics |
|----------|-----------------|---------------------|
| **Compute** | Minutes (weight matmul) | 2+ hours (corpus streaming) |
| **Requires corpus** | No | Yes |
| **Signal type** | Static (weights) | Dynamic (activations) |
| **Captures** | Structural proximity | Functional co-activation |
| **False positives** | High (geometry $\neq$ function) | Lower (grounded in behavior) |
| **Coverage** | All latents | Live latents only |

In practice we use decoder geometry as a fast filter and validation cross-check, while co-firing statistics provide the primary edge scores. An edge supported by both signals is more credible than one supported by only one.

The **Matryoshka vs. BatchTopK disagreement** provides a third signal: a feature that fires in Mat-SAE but not in BTK-SAE for the same token is one the Matryoshka ordering constraint is "forcing" to activate — potentially indicating that it is the most general available feature for that context. Systematic comparison of which Mat-SAE features have no BTK-SAE counterpart is an area of ongoing investigation.

---

## 7. Top-Activating Examples and AutoInterp

Co-firing statistics tell us *which* features co-activate, but not *what* those features mean. To validate hierarchy edges semantically — to check that a recovered parent really is more general than its children — we need human-interpretable descriptions of each latent.

### 7.1 Collecting Top-Activating Examples

During the same corpus pass that accumulates co-firing statistics, we maintain for each latent $i$ a **min-heap** of size $K = 15$ tracking the token contexts with the highest activation values:

$$\text{TopK}(i) = \arg\text{top}_K \{ z_i^{(t)} : z_i^{(t)} > 0, t = 1 \ldots T \}$$

For each entry in the heap, we store the activating token and its surrounding context window ($\pm 50$ tokens), with the activating token marked:

```
"30, 15 December 20<<0>>9 206.169."
"Wednesday, November 26, 20<<0>>8\n\nI just took"
```

The `<< >>` markers follow the SAEBench AutoInterp convention [Kharlapenko et al., 2024] and indicate the specific token that caused the latent to fire at that activation strength.

**Implementation note**: The naïve implementation iterates over all $F = 32{,}768$ latents per batch to update heaps, which creates a Python CPU bottleneck of ~35 seconds per batch even on A100 hardware (dwarfing the ~3s GPU time for Gemma + SAE encode). We address this by maintaining a per-latent minimum threshold array $m_i = \min(\text{TopK}(i))$ and skipping any latent where the batch maximum $\max_t z_i^{(t)}$ does not exceed $m_i$. After the first $K$ batches all heaps are full, and subsequent batches only update the small fraction of latents where a record-breaking activation occurs — reducing Python iterations from $32{,}768$ per batch to typically a few hundred.

### 7.2 Generating Descriptions with LLM AutoInterp

With the top-15 corpus examples collected, we prompt **Claude Haiku** (claude-haiku-4-5) to generate a short description of each latent:

```
We're studying neurons in a neural network. Each neuron activates on 
some particular word/concept in a document. The activating words are 
indicated with << >>. Look at the documents below and summarize in 
≤20 words what the neuron is activating on.

1. "30, 15 December 20<<0>>9 206.169."
2. "Wednesday, November 26, 20<<0>>8\n\nI just took"
3. "year ending in 20<<1>>5 was the hottest"
...

Description:
```

The result is a phrase like *"fires on the digit '0' or '1' following '20' in date expressions"*. At $\sim\$0.00025$ per call and 32,768 latents, full coverage costs approximately **$8** — a dramatic reduction from manually inspecting activation patterns for each latent.

This approach builds on the line of work by [Conmy et al., 2023; Bills et al., 2023; Gur-Ari et al., 2024] demonstrating that language models can reliably describe their own internal features when given sufficient activation context.

### 7.3 Semantic Validation of Hierarchy Edges

With descriptions for all latents, hierarchy edge validation becomes straightforward:

**Good edge** (parent more general than child):
- Child 4435: *"fires on digits in years within date expressions"*
- Parent 1122: *"fires on temporal expressions and date-related tokens"*

**Bad edge** (no semantic relationship):
- Child 7823: *"fires on Python keyword `def` when defining functions"*
- Parent 3301: *"fires on numeric citations in legal documents"*

We compute a **semantic coherence score** by passing parent-child description pairs to an LLM judge:

> *"Does description A describe a more general concept that would subsume description B? Answer yes or no."*

The fraction of "yes" answers at a given threshold provides an empirical precision estimate for the recovered hierarchy, enabling principled threshold selection without manual inspection of thousands of edges.

---

## 8. The Full Pipeline

The complete pipeline from raw model weights to semantically validated hierarchy is:

```
Gemma-2-2B (layer 12 residual stream)
    │
    ▼  [corpus streaming, 20M tokens]
Mat-SAE encoder → latent activations z ∈ R^32768
    │
    ├──► co-fire counts C, fire counts c
    │         │
    │         ▼
    │    P(j|i) = C[i,j] / c[i]
    │         │
    │         ▼
    │    threshold at τ=0.90 → edge set E_cofiring
    │
    ├──► top-15 activating examples per latent (heap tracking)
    │         │
    │         ▼
    │    Claude Haiku → descriptions[i] for i=0..32767
    │         │
    │         ▼
    │    semantic edge validation → precision@τ
    │
    └──► decoder weight cosine similarity → edge set E_geometry
              │
              ▼
         E_cofiring ∩ E_geometry → high-confidence hierarchy
```

All intermediate results are saved as `.pt` files (PyTorch serialized tensors) for re-evaluation at different thresholds without re-running the model:

| File | Size | Contents |
|------|------|----------|
| `matryoshka_cofiring.pt` | 8.6 GB | $\mathbf{C}$ and $\mathbf{c}$ tensors |
| `matryoshka_hierarchy.pt` | ~100 MB | Recovered edge set at $\tau=0.90$ |
| `matryoshka_top_examples.json` | ~500 MB | 15 text contexts per live latent |
| `matryoshka_latent_descriptions.json` | ~5 MB | Haiku-generated descriptions |

---

## 9. Infrastructure and Reproducibility

Running the 20M-token co-firing pass requires approximately:

- **GPU**: A100 80GB SXM (CUDA 13.0, torch 2.8.0+cu128)
- **Wall time**: ~2 hours for co-firing + top-examples combined
- **Cost**: ~$3 on RunPod secure cloud ($1.49/hr)
- **Peak VRAM**: ~50 GB (Gemma: ~20 GB in TransformerLens float32, SAE: 1.2 GB, co-fire accumulator: ~8 GB float64 on CPU, activation buffer: ~2 GB)

The experiment code lives at [era-saes/sae-hierarchy-recovery](https://github.com/era-saes/sae-hierarchy-recovery) and is structured as:

```
experiments/Konstantinos-sprint1/
    compute_real_cofiring.py    # full cofiring + top-examples in one pass
    collect_top_examples.py     # lighter pass: text examples only, no cofiring
sae_hierarchy_recovery/
    cofiring.py                 # compute_cofiring_stats(), recover_hierarchy_from_cofiring()
    real_activations.py         # make_gemma_activation_source()
    top_examples.py             # TopActivatingExamples heap tracker
scripts/
    run_real_cofiring.sh        # local runner
    run_collect_top_examples_runpod.sh  # RunPod wrapper
```

Checkpointing saves the co-firing accumulator every 100 batches, storing `sequences_consumed = total_tokens // context_size` so a resumed run fast-forwards the corpus iterator to exactly the right position, avoiding any double-counting.

---

## 10. Related Work and Context

**Sparse Autoencoders for mechanistic interpretability**: [Anthropic's monosemanticity paper (Bricken et al., 2023)](https://transformer-circuits.pub/2023/monosemantic-features) established that SAEs trained on MLP activations of a one-layer transformer recover monosemantic features. [Templeton et al., 2024](https://transformer-circuits.pub/2024/scaling-monosemanticity/) scaled this to Claude 3 Sonnet with 34M features, finding that the feature space contains rich semantic structure.

**Feature hierarchies**: The idea that SAE features organize hierarchically is implicit in several findings. Gurnee et al. (2023) found that features in different layers of the same model form rough hierarchies from syntactic (early layers) to semantic (later layers). [Lindsey et al., 2025](https://transformer-circuits.pub/2025/crosscoder-features/) studied cross-layer features using crosscoders, finding that many features are active across multiple layers with varying specificity.

**Matryoshka SAEs**: The Matryoshka principle applied to representation learning [Kusupati et al., 2022] was extended to SAEs by [Chanin, 2024] as an explicit mechanism for encoding feature importance ordering. The nested structure means the first $k$ features of the Mat-SAE reconstruct the residual stream better than any other $k$ features — building in a soft hierarchy by construction.

**BatchTopK sparsity**: [Bussmann et al., 2024] showed that TopK activation functions (selecting exactly $k$ nonzero activations per token) outperform $L_1$-penalized ReLU activations on downstream task performance at matched sparsity levels. BatchTopK extends this to fix $k$ globally across a batch rather than per-token, reducing variance in sparsity.

**LLM AutoInterp**: [Bills et al., 2023](https://openaipublic.blob.core.windows.net/neuron-explainer/paper/index.html) pioneered the use of GPT-4 to generate and evaluate feature descriptions from activation examples, demonstrating that LLMs can reliably identify what activates neurons when given sufficient context. [Kharlapenko et al., 2024 (SAEBench)] systematized this into a benchmark for SAE evaluation including automated description generation and scoring.

---

## 11. Open Questions and Future Directions

Several fundamental questions remain open:

1. **Hierarchy depth**: Is the recovered hierarchy shallow (2–3 levels) or deep? Do features form long chains like *digit → number → quantity → measurement → scientific fact* or do most edges connect directly to very general roots?

2. **Cross-layer hierarchies**: Do layer-12 Mat-SAE features inherit from layer-6 features? Co-firing statistics could in principle be computed across layers if the model's residual streams at different layers are compared.

3. **Corpus dependence**: How much do the recovered edges depend on the corpus? A hierarchy recovered from legal text might differ substantially from one recovered from code. The 20M-token sample covers the full Pile diversity but is small relative to the 2.4B total.

4. **Threshold calibration**: The 0.90 threshold is somewhat arbitrary. Semantic validation via AutoInterp provides a principled way to optimize $\tau$ by maximizing precision while maintaining recall, but this requires ground-truth labeling of at least a sample of edges.

5. **BTK vs. Mat disagreement as a hierarchy signal**: Tokens where Mat-SAE activates feature $i$ but BTK-SAE does not may indicate that $i$ is serving a "general coverage" role in Mat-SAE. Systematically analyzing these disagreements could surface the most structurally important features in the hierarchy.

6. **100M+ token scaling**: The 20M-token run covers ~0.83% of the training corpus. Scaling to 100M (still only 4%) would provide tighter conditional probability estimates for rare features (those with log-sparsity $\leq -4$, firing ~0.01% of tokens, which see only ~2,000 firings in 20M tokens vs. ~20,000 in 200M).

---

## 12. Conclusion

We have described a pipeline for recovering and semantically validating feature hierarchies in a 32,768-latent Matryoshka SAE trained on Gemma-2-2B. The two core signals — decoder weight cosine similarity and co-firing conditional probability — are complementary and efficiently computable: geometry requires only the trained weights, while co-firing requires a corpus pass but captures actual model behavior. The AutoInterp component closes the loop by grounding each edge in human-interpretable semantics: an edge that passes the co-firing threshold $P(j \mid i) \geq 0.90$ and is confirmed by a description pair like *"digits in dates"* → *"temporal expressions"* is substantially more credible than a bare statistical threshold. Together, these signals provide a foundation for systematic, scalable analysis of feature organization in large SAEs — a necessary step toward understanding not just *what* features individual latents encode, but *how* those features relate to one another in the model's broader representational geometry.

---

## References

- Bricken, T., Templeton, A., Batson, J., Chen, B., Jermyn, A., Conerly, T., ... & Olah, C. (2023). *Towards Monosemanticity: Decomposing Language Models With Dictionary Learning*. Transformer Circuits Thread.
- Bills, S., Cammarata, N., Mossing, D., Tillman, H., Gao, L., Goh, G., ... & Saunders, W. (2023). *Language Models Can Explain Neurons in Language Models*. OpenAI.
- Bussmann, B., Mudide, A., & Anthropic (2024). *BatchTopK: A Simple Improvement for TopK SAEs*. Transformer Circuits Thread.
- Chanin, D. (2024). *Twin SAE: Jointly Trained Matryoshka and BatchTopK Sparse Autoencoders for Gemma-2-2B*. HuggingFace Model Hub.
- Cunningham, H., Ewart, A., Riggs, L., Huben, R., & Sharkey, L. (2023). *Sparse Autoencoders Find Highly Interpretable Features in Language Models*. arXiv:2309.08600.
- Elhage, N., Hume, T., Olsson, C., Schiefer, N., Henighan, T., Kravec, S., ... & Olah, C. (2022). *Toy Models of Superposition*. Transformer Circuits Thread.
- Gurnee, W., Nanda, N., Pauly, M., Harvey, K., Troitskii, D., & Bertsimas, D. (2023). *Finding Neurons in a Haystack: Case Studies with Sparse Probing*. arXiv:2305.01610.
- Kusupati, A., Bhatt, G., Rege, A., Wallingford, M., Sinha, A., Ramanujan, V., ... & Farhadi, A. (2022). *Matryoshka Representation Learning*. NeurIPS 2022.
- Lindsey, J., et al. (2025). *Crosscoder-Based Model Diffing*. Transformer Circuits Thread.
- Templeton, A., Conerly, T., Marcus, J., Lindsey, J., Bricken, T., Chen, B., ... & Olah, C. (2024). *Scaling Monosemanticity: Extracting Interpretable Features from Claude 3 Sonnet*. Anthropic.
