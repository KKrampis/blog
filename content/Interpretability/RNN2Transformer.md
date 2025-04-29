---
title: "From RNNs to Transformers"
---

### A short history and review.

Neural sequence transduction models represent a cornerstone of modern natural language processing, moving beyond fixed-length input-output mappings to handle variable-length sequences – think machine translation, text summarization, or speech recognition.  Traditional approaches to these tasks relied heavily on recurrent neural networks (RNNs), particularly LSTMs and GRUs, to process sequential data. However, these RNN-based models suffered from limitations, notably the vanishing/exploding gradient problem hindering long-range dependency capture, and inherent sequential processing preventing parallelization. The breakthrough came with the introduction of the *sequence-to-sequence* (seq2seq) architecture, initially proposed by Sutskever et al. (2014) and further refined by Bahdanau et al. (2015), and subsequently revolutionized by the Transformer architecture.  At its core, a seq2seq model comprises two primary components: an *encoder* and a *decoder*. The encoder processes the input sequence and compresses it into a fixed-length *context vector* – a crucial bottleneck. The decoder then takes this context vector and generates the output sequence, one element at a time.  Let’s define some key mathematical elements. Given an input sequence  $X = (x_1, x_2, ..., x_{T_x})$ and an output sequence $Y = (y_1, y_2, ..., y_{T_y})$, the encoder maps $X$ to a sequence of hidden states $h = (h_1, h_2, ..., h_{T_x})$, where each $h_i$ represents the hidden state at time step *i*.  The decoder then uses these hidden states, along with the previously generated output tokens, to predict the next token in the sequence. The probability of generating the output sequence *Y* given the input sequence *X* is modeled as:

$P(Y|X) = \prod_{t=1}^{T_y} P(y_t | y_{<t}, X)$

where $y_{<t}$ represents all the previously generated output tokens up to time step *t*.  The initial work relied on the final hidden state of the encoder as the sole context vector. While conceptually simple, this proved to be a significant limitation, as all information from the input sequence had to be squeezed into a single vector, leading to information loss, particularly for longer sequences. This is where the *attention mechanism*, introduced by Bahdanau et al. (2015), became pivotal.  Attention allows the decoder to selectively focus on different parts of the input sequence when generating each output token. Instead of relying solely on the final hidden state, the decoder computes a *context vector* $c_t$ at each time step *t* as a weighted sum of the encoder hidden states:

$c_t = \sum_{i=1}^{T_x} \alpha_{ti} h_i$

where $\alpha_{ti}$ represents the attention weight assigned to the *i*-th encoder hidden state $h_i$ when generating the *t*-th output token. These attention weights are calculated using a *score function* $score(h_i, s_{t-1})$, where $s_{t-1}$ is the decoder hidden state at the previous time step. The score function measures the relevance of the *i*-th encoder hidden state to the current decoder state. Common choices for the score function include dot product, scaled dot product, and additive attention (also known as Bahdanau attention).  For instance, the additive attention mechanism is defined as:

$e_{ti} = v^T \tanh(W_1 h_i + W_2 s_{t-1})$

$\alpha_{ti} = \frac{exp(e_{ti})}{\sum_{j=1}^{T_x} exp(e_{tj})}$

where $W_1$ and $W_2$ are weight matrices, and *v* is a weight vector. The resulting attention weights are then normalized using a softmax function to ensure they sum to 1. The context vector $c_t$ is then concatenated with the decoder hidden state $s_{t-1}$ and used to predict the next output token.



The Transformer architecture, introduced by Vaswani et al. (2017), took this a step further by *entirely* abandoning recurrence and relying solely on attention mechanisms. It introduces *self-attention*, allowing the model to relate different positions of the *same* input sequence to compute a representation of the sequence.  The core of the Transformer is the *multi-head attention* mechanism, which allows the model to attend to different parts of the input sequence with different learned linear projections.  Mathematically, multi-head attention computes multiple attention outputs in parallel and concatenates them. Let $Q$, $K$, and $V$ represent the query, key, and value matrices, respectively.  The attention weights are calculated as:

$Attention(Q, K, V) = softmax(\frac{QK^T}{\sqrt{d_k}})V$

where $d_k$ is the dimension of the key vectors. The scaling factor $\sqrt{d_k}$ prevents the dot products from becoming too large, which can lead to vanishing gradients.  The multi-head attention then computes *h* parallel attention outputs and concatenates them:

$MultiHead(Q, K, V) = Concat(head_1, ..., head_h)W^O$

where $head_i = Attention(QW_i^Q, KW_i^K, VW_i^V)$ and $W_i^Q$, $W_i^K$, $W_i^V$, and $W^O$ are learned weight matrices.  The Transformer also utilizes *positional encoding* to incorporate information about the order of tokens in the input sequence, as the attention mechanism is permutation-invariant.  These models, with their ability to parallelize computation and capture long-range dependencies, have become the foundation for state-of-the-art performance in a wide range of sequence transduction tasks, and form the basis of models like BERT, GPT, and many others. They represent a significant advancement over traditional RNN-based approaches, offering improved performance, scalability, and interpretability.



