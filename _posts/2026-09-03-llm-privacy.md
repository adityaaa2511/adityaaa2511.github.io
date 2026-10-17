---
layout: post
title: "Can LLMs leak private data?"
date: 2026-09-03 21:00:00 +0530
tags: [AI safety, LLM privacy, Text Sanitization, Adversarial Attacks, Differential Privacy]
comments: false
_styles: >
  .post-title { font-weight: 500; margin-bottom: 0.75rem; }
  .post-meta { margin-top: 0.6rem; margin-bottom: 0.5rem; }
  .post-content h3, .post-content h4 { font-weight: 500; margin-top: 2rem; margin-bottom: 0.75rem; }
  .post-content p { line-height: 1.75; margin-bottom: 1.25rem; }
---

### Background

AI safety has gained significant traction in the last couple of years and this post serves as a  part of a comprehensive effort to broaden my knowledge of this field. Recent research shows that adversaries can extract training data from LLMs, infer Personal Identifiable Information (PPIs like:- social security number, address, dob, gender, occupation etc.) from training data and user queries, reconstruct system prompts which serve as valuable IPs for companies which can have catastrophic consequences. This post details how adversaries can maliciously attack LLMs for extracting sensitive data and some strategies for mitigating such attacks.

### Defense Mechanisms and Mitigation Strategies

#### Differential Privacy

From the list of theoretically provable methods for preventing private data leakage, training models with Differential Privacy (DP) is the de-facto standard. The standard algorithm for this is DP-SGD which involves clipping the per-sample gradient norms to ensure bounded contribution from each data point and further adding gaussian noise to each per-sample clipped gradient during loss aggregation so that the contribution from a data point cannot be traced back using its gradients. The clipping threshold is a hyperparameter and the standard distribution of the gaussian distribution is selected such that the algorithm satisfies (ε, δ)-Differential privacy. ε is the privacy budget which bounds how much a distirbution can shift if one record is added or removed  and δ is the failure probability accounting for a small chance that the ε-bound breaks.

#### Differentially Private Fine-tuning

However straightforward approaches for leveraging DP-SGD during training have resulted in large performance drops on downstream NLP tasks and large computational overheads. [3](#reference-3) propose a meta-framework for utilizing DP with fine-tuning for mitigating the aformentioned issues. Specifically, the authors suggest augmenting extra parameters to the base model whose parameters remain frozen while the additional parameters are fine-tuned on a small, private dataset for the downstream task. This parameter-efficient approach works well under privacy constraints because the fine-tuned weights remain on a low-dimensional manifold close to the pre-trained weights, meaning the noise injected by DP-SGD does not destroy the knowledge learned during pre-training. 

[2](#reference-2) suggest that using non-trivial hyperparameters for fine-tuning with DP and improving downstream task alignment with pre-training objective can result in improvement in performance, even to the point of matching strong non-private methods. Contrary to traditional fine-tuning, using large batch sizes and large learning rates seem to achieve better results when the no. of training epochs is fixed (compute budget is constant). When the no. of gradient step updates is fixed, increasing the batch size is almost always beneficial at a fixed privacy budget. Further, using a small clipping norm to ensure that most gradients get clipped also leads to better performance.

The authors also indicate that aligning the fine-tuning task of classification with the standard pretraining objective by modelling it as text infilling in :- "[INPUT]. It is [MASK].", the options being [AWESOME] and [TERRIBLE], leads to a 3-5% increment in evaluation metrics.

#### Ghost Clipping

The paper also describes a memory reduction technique called **Ghost Clipping** which alleviates the issue of instantiating a gigantic per-sample gradient matrix and instead leverages the input and activation gradients from a linear layer (both are available during the backward pass) to calculate the gradient norms

Let:

- $$B$$ = batch size, $$T$$ = sequence length (time steps)
- $$a \in \mathbb{R}^{B \times T \times d}$$ = the input to the linear layer
- $$W \in \mathbb{R}^{p \times d}$$ = the weight matrix
- $$g \in \mathbb{R}^{B \times T \times p}$$ = the gradient of the loss with respect to the layer's output

The actual per-example gradient for the $$i$$-th example is the product of two matrices (using chain-rule):

$$\nabla_W \mathcal{L}_i = g_i^\top a_i \in \mathbb{R}^{p \times d}$$

Instead of computing this $$p \times d$$ matrix, Ghost Clipping uses a mathematical identity to compute its squared Frobenius norm ($$||\nabla_W \mathcal{L}_i||_F^2$$) using only the inner products of the sequence dimensions:

$$||\nabla_W \mathcal{L}_i||_F^2 = \text{vec}(a_i a_i^\top)^\top \text{vec}(g_i g_i^\top)$$

By rewriting the equation this way, the memory complexity drops from O(Bpd) (the size of the explicit gradient matrix) to O(BT^2) (the size of the $$aiai^T$$ and $$gigi^T$$ matrices).

Now the per-example clipping coefficient $$c_i$$ can be computed cheaply from this norm:

$$c_i = \min\!\left(1,\, \frac{C}{||\nabla_W \mathcal{L}_i||_F}\right)$$

where $$C$$ is the clipping threshold. Instead of clipping each per-example gradient matrix explicitly, the backward pass is then run on the reweighted loss:

$$\sum_{i=1}^{B} c_i \mathcal{L}_i$$

Because the gradient of this reweighted loss with respect to $$W$$ is exactly $$\sum_i c_i \nabla_W \mathcal{L}_i$$ — the correct sum of clipped gradients — Ghost Clipping avoids ever materialising the full $$p \times d$$ per-example gradient matrix while producing an identical result to standard DP-SGD clipping.

#### Dynamic Text Sanitization at Inference Time

The techniques mentioned above are best suited while pretraining LLMs or fine-tuning them for a downstream task. However, there exists research that shows LLMs can still leak sensitive information from user prompts and thus, there is a need to ensure privacy at inference time. [4](#reference-4) proposes DYNTEXT - Dynamic Text Sanitization which employs semantically-aware density information under DP for constructing a dynamic adjacency list for each token and then using a replacement mechanism for sanitization.

The size of the list is dynamically adjusted using the calculated density information. The density information is first normalization using min-max to preserve the distribution shape and other statistical properties:- 

$$ \hat{F}(t) = \frac{F(t) - F_{\min}}{F_{\max} - F_{\min}} $$

Then the size is calculated as:-

$$ k_t = \max\left(1, \left\lfloor (1 - \hat{F}(t))K \right\rfloor\right) $$

where K is the maximum allowed size and is a hyperparameter.

The replacement mechanism first calculates a similarity score based on the euclidean distance between the token and its constituents from the adjacency list as follows:- 

$$u(t, \hat{t}) = 1 - \frac{d(t, \hat{t})}{d_{\max}}$$

where $$d_{\max} = d(t, t_{k_t}) = \max_{\hat{t} \in C_r(t)} d(t, \hat{t})$$ is the semantic distance to the farthest token in the adjacency list $$C_r(t)$$.

Further, given the privacy budget epsilon_r of the replacement mechanism, a probability distribution over the candidate token is computed as follows:-

$$Pr[M(t) = \hat{t}] = \text{softmax}\left(\frac{\varepsilon_r \cdot u(t, \hat{t})}{2\Delta u}\right)$$

The similarity score is used to assign a higher priority to candidates that are more semantically similar to the token being replaced while a lower privacy budget can be enforced to strengthen privacy protection.

Now, lets get into how this density information is calculated for each token.

In the embedding space, the authors propose to compute the K-closest distance between the current tokeen and all other tokens as:-

$$
d(t, t_K) = \left\|\phi(t) - \phi(t_K)\right\|_2
$$

where phi maps each token to their vectors in the embedding space. Then, a threshold is calculated as the average of this K-th closest distance across all tokens. Finally, the desnity information for each token is the number of tokens whose euclidean distance with the current token falls within this threshold.

$$ f(t) = \left|\left\{\hat{t} \in V_t \mid d(t, \hat{t}) \leq \gamma\right\}\right| $$

To prevent this density information fro leaking sensitive information about tokens (i.e semantic desnity), gaussian noise is added to achieve differential privacy.

$$ F(t) = f(t) + \mathcal{N}(0, \sigma_d^2) $$

The variance of the gaussian noise is determined from the privacy-budget parameters and the sensitivity.

$$ \sigma_d^2 = \frac{2(\Delta f)^2 \ln(1.25/\delta)}{\varepsilon_d^2} $$

Further, to mitigate information leakage from sensitivty flunctuations and noise amplitudes, a token-level smoothing is also proposed. They describe existing methods to calculate token sensitivity which fall under Global and Local methods. The local and global sensitivity of a query function (in this case, the density information) is :- 

$$
LS_f(t) = \max_{\hat{t} \in C_r(t)} \left|f(t) - f(\hat{t})\right|
$$

$$
GS_f = \max_{t \in V_t}\left(LS_f(t)\right)
$$

Individually applied, both have their limitations. GS adds exessive noise as it is based on worst-case estimate across all tokens and while LS adds noise based on each token, the noise amplitude can leak privacy information about the token. The authors employ a smoothed version of LS to overcome this issues. Specifically, they use a sensitivity parameter $$
S_{f,\beta}(t) $$ when adding noise to the token desnity information. For a token t and t's adjacent token $$ \hat{t} \in C_r(t)$$:-

$$
S_{f,\beta}(t)
=
\max_{\hat{t} \in C_r(t)}
\left(
LS_f(\hat{t}) \cdot e^{-\beta d(t,\hat{t})}
\right)
$$

Since $$ S_{f,\beta}(t) $$ is the maximum value of $$ LS_f(\hat{t}) $$, it is more sensitive to changes in local tokens allowing it to better preserve semantic information. Moreover, $$ \beta $$ can be used to regulate privacy. A larger $$ \beta $$ can decrease local sensitivity and thereby, preserve semantic information for tokens in high density regions by reducing the influence of distant tokens and a smaller $$ \beta $$ can allow distant tokens to contribute more for tokens in low density areas, enhancing privacy protection. 

### Text Sanitization 

Text Sanitization is the process of replacing sensitive data like private information (age, address, gender, occupation, family/health info etc.) with relatively less revealing tokens while maintaining text coherence and utility for downstream tasks.

#### Zero-Shot Redaction and Substitution

The first approach beyond rule-based processing utilizes a zero-shot redaction and subsitution framework as proposed by [5](#reference-5). First, the document is preprocessed into tokens using a tokenizer consistent with the downstream LLM which is used for calculating the probability of occurence of each token by using Masked language modelling. If at a particular position, the probability of that tokens occurance is less than a threshold (set according to privacy requirements), then that token is flagged for substitution. In case a word consists of multiple tokens, the method starts by masking the suffix token and subsequently calculates the probability by moving towards the prefix. The final probability is the product of all such values.

Substitution for numerical tokens proceeds by replacing them with a value of similar length and for non-numerical tokens, the N most probable tokens at that position are considered using the probability distribution given by the LLM. These are sorted in decreasing order according to their euclidean distance (to preserve similarity) and a candidate token is randomly selected from the top-K tokens within a threshold s. If there are no such candidate tokens, the token is dropped. A substitution table is maintained to ensure consistent replacement if a similar token is encountered later on.

This is certainly not an optimal method as a forward pass is required over each token which results in high latency and depends heavily on the prob. dist. given by the LLM. Niche terms and rare jargon might be falsely flagged and high prob. sensitive terms (like:- names, age) might evade the filtering criteria. Substitution based on embedding similarity also doesnt gaurentee privacy.

#### Self-Disclosure Detection and Text Abstraction

The next approach for text anonymization could be to fine-tune an LLM to detect potentially sensitive information and then employ some sort of abstraction techniques in order to mask the sensitive information. However, for fine-tuning the LLM, we required annotated data. [6](#reference-6) solves this issue by proposing a detailed taxonomy of 19 self-disclosure categories obtained from public Reddit data and assigning labels to each instance of private information in this self-disclosure data. They select spans with contextual information instead of a single word for more nuanced training signals. For example, “I live in the US” would be preferred over a minimal span like “US”, which is isolated from its self-referential context. 

Text abstraction is performed using another fine-tuned LLM to reduce the specific details of the content while preserving the downstream task utility. Three methods are proposed for this:- Sentence paraphrasing, Sentence abstraction (w/o providing spans and w/ spans) and span abstraction. Ablations show that sentence abstraction, which might also mask non-disclosure text thereby affecting the original style and text utlity, achieves similar performance to span abstraction when disclosure spans are provided.

Fine-tuning is further performed using three methods:- 

1) **Sampling:** Three abstractions are sampled from the LLM seperately.

2) **End-to-End:** The abstraction model is trained to generate three abstraction in one response.

3) **Iterative Generation:** An iterative approach which breaks the three abstractions (A,B,C) as 3 seperate training instances- (input->A), (input->A->B), (input->A->B->C).

Results indicate that 2) and 3) achieve the best performance when trained without CoT before answering. This may be because the goal is to perform simple text abstraction and not to solve complex problem for which multi-step reasoning is ill-suited. Elaboration and justification can reduce performance on metrics like:- BLEU, ROUGE and human-ranking which value directness and conciseness.

#### Inferential Feedback for Text Sanitization

As shown by approaches in the attack section, modern LLMs can be misused for high-accuracy predictions
of personal attributes from seemingly non-revealing online posts. Existing methods for text anonymization are based on rules and named-entity recognition which don't account for inference derived from reasoning. As such, [7](#reference-7) propose to utilize the increasing inferencing capabilties of LLMs and employ them for text sanitization. Concretely, they present a framework in which an adversarial LLM is used to infer a set of attributes iteratively. The set of attributes that we want to obfuscate along with the text are sent to the adversarial LLM which gives feedback on which attributes it can infer from the current text. This feedback plus the text are sent to the anonymizer LLM which tries to adapt relevent spans according to the feedback. In round i, only the text sanitizated after round i-1 is given to the anonymizer and not all versions of modified text. When the adversarial LLM cannot infer anymore attributes or after a fixed number of rounds, this cycle of inferential feedback and sanitization is halted.

The papers uses an LLM-as-a-judge to measure the downstream task utility and readability along with traditional metrics like:- BLEU and ROUGE-1 for evaluations. Ablations suggest that providing adversarial feedback is crucial for ensuring higher privacy and utility, which makes considerably larger improvements per round as compared to the case where adversarial feedback is not provided. 

#### Utility-Preserving Text Anonymization

A concern about the previous approach is that though adversarial feedback is helpful for obfuscation and privacy-preservation, the downstream task-utility is severly hampered due to over anonymization. For mitigating this issue, a robust utility-preserving framework is proposed by [8](#reference-8). 

The problem is formulated as Lexicographic optimization involving 2 competiting objectives:- privacy enhancement and utility preservation. Privacy enhancement is the primary objective; the framework prioritizes achieving a higher degree of privacy protection and only after that task utility preservation is optimized. Formally, a solution \(x_a \in X_0\) is lexicographically preferable to another solution \(x_b \in X_0\), denoted by \(x_a \succ_{\mathrm{lex}} x_b\), if and only if:

$$
f_p(x_a) > f_p(x_b)
\quad \text{or} \quad
\left(
f_p(x_a) = f_p(x_b)
\;\text{and}\;
f_u(x_a) > f_u(x_b)
\right)
$$

where \(f_p(\cdot)\) and \(f_u(\cdot)\) denote the privacy and utility objective functions, respectively and \(X_0\) denotes the set of all possible edits of \(x_0\).

The framework has 3 components:-

1) **Privacy Evaluator**:- An LLM instance whose job is assess the privacy protection levels, ensuring that private content is adequately obscured against re-identification. It generates top-K re-indentification results which are compared with their ground-truth label. If a match is found, the rank of the matched result is considered a privacy protection score \(p_t\). Further, the evaluator is prompted to provide natural language feedback \(f_t\), detailing the clues that led to the correct inference. If there is no match then \(p_t\) = K+1.

2) **Utility Evaluator**:- An LLM instance whose responsibility is to examine the downstream task utility of the anonymized text. Given the ground truth label c, the utility score \(u_t\) can be calculated using its logit following traditional uncertainty quantification methods.

3) **LLM Optimizer**:- To ensure that the primary objective of achieving maximum privacy is prioritized over utility, the lexicographic-optimizer LLM operates in two different modes. When the privacy objective value has not yet reached the pre-set maximum K + 1, the lexicographic optimizer should focus on maximizing the privacy objective, which is achieved by taking the value of the meta-instruction variable as \(I_{pr}\) that instructs the LLM to further anonymize \(x_t\) according to textual feedback \(f_t\).

Once the privacy objective value has reached the maximum threshold, the meta instruction shifts to \(I_{ur}\), prompting the LLM to optimize the utility level without compromising the achieved privacy objective value.

The iterative process continues until either the pre-defined maximum values for both objectives
are reached or the maximum number of iterations T is met.

**Evaluation Metrics**:- The evaluation focuses on two critical aspects: disclosure risk and utility preservation. Disclosure risk is assessed by measuring the Success Rate (SR) of a state-of-the-art LLM in inferring personal information from anonymized text. Besides this, the LLM is prompted to generate the Confidence Scores (CS), evaluating the degree of confidence with which anonymized text can be linked to the ground-truth personal information.

Utility preservation metrics are gauged by the performance of a BERT-based classifier finetuned on original train data and tested on the test data that is anonymized by RUPTA and other methods, including Accuracy, Precision, Recall, F1 Score, and the classifier’s loss function value indicating classification uncertainty. 

<hr style="margin-top: 2.5rem; margin-bottom: 2.5rem; border: none; border-top: 1px solid var(--global-text-color-light);">

### References

1. <span id="reference-1"></span>Abadi, M. et al. (2016). [Deep Learning with Differential Privacy](https://arxiv.org/pdf/1607.00133)
2. <span id="reference-2"></span>Li, X., et al. (2022). [Large Language Models Can Be Strong Differentially Private Learners](https://openreview.net/forum?id=bVuP3ltATMz).
3. <span id="reference-3"></span>Yu, D. et al. (2022). [Differentially Private Fine-tuning of Language Models](https://openreview.net/forum?id=Q42f0dfjECO).
4. <span id="reference-4"></span>Zhang, J. et al. (2025). [DYNTEXT: Semantic-Aware Dynamic Text Sanitization for Privacy-Preserving LLM Inference](https://aclanthology.org/2025.findings-acl.1038/).
5. <span id="reference-5"></span>Albanese, F. et al. (2023). [Text Sanitization Beyond Specific Domains: Zero-Shot Redaction & Substitution with Large Language Models](https://arxiv.org/abs/2311.10785).
6. <span id="reference-6"></span>Dou, Y. et al. (2024). [Reducing Privacy Risks in Online Self-Disclosures with Language Models](https://arxiv.org/abs/2311.09538)
7. <span id="reference-7"></span>Staab, R. et al. (2025). [Language Models are Advanced Anonymizers](https://arxiv.org/abs/2402.13846)
8. <span id="reference-8"></span>Yang, T. et al. (2025). [Robust Utility-Preserving Text Anonymization Based on Large Language Models](https://aclanthology.org/2025.acl-long.1404/)

