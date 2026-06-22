---
title: "Number Understanding in LLMs"
---

We will first look at a 2024 [study](https://arxiv.org/abs/2410.11781) by Levy and Mor, were it
was investigated how Large Language Models (LLMs) handle numerical tasks by assessing whether these
models internally process the digits of numbers similarly to other characters, as opposed to comprehending
actual numeric values. Their findings indicated that numerical errors made by LLMs in
addition problems like "132+238+324+139", which sums to 833, tend to reflect digit similarity
errors such as "633" or "823", rather than errors closer in numeric value like "831" or "834".
This pattern suggests that LLMs are more inclined to make errors that are closer in 'digit
representation space' rather than in numerical value, highlighting a unique aspect of how
numbers are encoded.

The authors used [linear probes](https://nlp.stanford.edu/~johnhew/interpreting-probes.html)
and found that the models encode numbers using circular representations in base 10 rather than
single digit representations. This research was conducted with the "Llama 3" model, with
8 billion parameters, tested with 5,000 queries to explore the nature of its numerical
inaccuracies. The study also evaluated the model on number comparison tasks, with number pairs
ranging from 0 to 999, each differing by a single digit in units, tens, or hundreds place.
The aim was to ascertain if errors were more influenced by specific differing digit positions
than by numerical proximity. Analysis demonstrated that errors frequently aligned with
multiples of 10 or 100 and with digit discrepancies rather than strictly numerical
errors, further underscoring the influence of digit representation on the model's mistakes.

The goal of linear probing is to determine how effectively number digits can be
predicted from the hidden representations captured by the model. This involves training what
are known as digit-wise probes, which attempt to decode the values of each digit of a number
from its hidden state. Consider a pre-trained transformer-based language model, denoted as $N$,
with $K$ layers and a hidden dimension of size $d$. In transformer-based language models, the
architecture is typically characterized by multiple layers of transformations applied to the
input data, which is referred to as having $K$ layers. Each layer processes information by
transforming it through mechanisms like [attention mechanism](https://en.wikipedia.org/wiki/Attention_mechanism),
resulting in representations that capture different aspects and complexities of the input.
Within each layer, the model maintains a hidden state vector for each position / token in the input
sequence, which is referred to in this context by a dimension size $d$. This dimension is essentially
the size of the vector, which numerically encodes features of the input token at a detail level
specific to that layer.

The hidden state of the $m$-th input token at layer $K$ is denoted as
$\mathbf{u}^k_m$ . This vector $\mathbf{u}^k_m$ is a snapshot of how the model represents a particular
token (the $m$-th in the sequence) at that specific stage in the model training or inference pipeline,
incorporating both the immediate context and the semantic features the model has identified up to that
layer for that specific toke. Each layer thus refines and transforms its input hidden states to
produce the next layer's states, progressively building a richer representation that influences
subsequent representation for each token.

Returning to the contrusted probes in the Levy and Mor study, they were designed to predict the numeric
value of each digit $j$ from the hidden representation $\mathbf{u}^k$. The optimization objective for this
process is given by:

\[
\mathbf{R}_{j,p}^k = \arg\min_{\mathbf{R}'' \in \mathbb{R}^{2 \times d}} \sum_{\langle
\mathbf{u}^k, y_j \rangle \in \mathcal{E}^k} \left\| \mathbf{R}'' \mathbf{u}^k -
\text{circular}_p(y_j) \right\|_2^2
\]

Here, $\mathcal{E}^k$ refers to a training set containing pairs
$\langle \mathbf{u}^k, y_j \rangle$, where $\mathbf{u}^k$ denotes the hidden representation
at layer $k$, and $y_j$ signifies the $j$-th digit of the number in consideration. The function
$\text{circular}_p(t)$ is defined as:

\[
\text{circular}_p(t) = \left( \cos\left(\frac{2\pi t}{p}\right), \sin\left(\frac{2\pi t}{p}\right) \right)
\]

This function maps a digit in base $p$ to a point on the unit circle, effectively encoding
the digit in a circular fashion. The circular probe, defined by $\mathbf{R}_{j,p}^k$, seeks
to find a transformation that minimizes the Euclidean distance between the transformed hidden
representation $\mathbf{R}'' \mathbf{u}^k$ and the circular embedding of $y_j$.
Using this set of circular probes at a given layer $k$, one constructs predictions for each
digit independently by applying:

\[
\text{digit}_{j,p}^k(\mathbf{u}^k) = \frac{p}{2\pi} \cdot \text{atan2}(\mathbf{R}_{j,p}^k
\mathbf{u}^k)
\]

This function converts the circular probe outputs back to a numeric value in base \( p \) using
the arctangent function, \(\text{atan2}\), which calculates the angle given the cosine and sine
values. By concatenating these digit predictions, we reconstruct the entire numeric value from
its hidden representation, effectively piecing together each digit’s contribution to represent
the full number.

Together, these formulations enable us to examine the efficacy of different base
representations in capturing digit information from the hidden states of LLMs. The outcome of
such probing experiments reveals insights into whether numbers are represented in value space
or if there exists a digit-wise encoding in the model's internal computations. Through this
method, the authors analyze how accurately a model's hidden states can be decoded to extract
individual digit values, ultimately confirming the base-10 representation hypothesis.

LITERATURE CITED:

Levy, Amit Arnold, and Mor Geva. 2024. "Language Models Encode Numbers Using Digit Representations in Base 10." arXiv, October 2024. https://arxiv.org/abs/2410.11781.
