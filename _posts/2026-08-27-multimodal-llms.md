---
layout: post
title: "Evolution of Vision Language Models"
date: 2026-08-27 12:00:00 +0530
description: My understanding of some foundational papers in Multimodal LLMs
tags: [Multimodal LLMs, VLMs]
comments: false
_styles: >
  .post-title { font-weight: 500; margin-bottom: 0.75rem; }
  .post-meta { margin-top: 0.6rem; margin-bottom: 0.5rem; }
  .post-content h3, .post-content h4 { font-weight: 500; margin-top: 2rem; margin-bottom: 0.75rem; }
  .post-content p { line-height: 1.75; margin-bottom: 1.25rem; }
---

### Background

Foundational models like:- ChatGPT, Claude and Gemini are inherently multimodal due to their capacity to process different types of inputs like:- images and audio along with standard text. In an effort to develop a deep understanding of the technology that goes into making these models, I am presenting my interpretation of some foundational papers. Hope you find it informative and interesting ! 

### 1. Learning Transferable Visual Models from Natural Language Supervision (Radford et al. - Feb 2021)

This paper attempts to address the prevalant limitation of computer systems at that time to predict only a fixed set of categories on which system is trained. The authors say that this limits the generalizability and usability of such systems and learning directly from text associated with images is promising research direction to alleviate this limitation. So, they demonstrate that a simple pre-training task of associating which (image,text) pairs go together is an efficient and scalable way to learn image representations which can enable zero-shot downstream tasks. 

#### 1.1 Contrastive Pre-training Objective

The obvious approach of associating (image,text) pairs is to jointly train an image feature extractor and a text transformer from scratch to predict the caption of the image. But, the authors suggest that this proves to be a difficult and inefficient task. 

Instead, they try to explore a potentially easier proxy task of pairing which (image,text) pair go together. Initially, the model (named CLIP) encodes N images into visual embeddings using a visual encoder (ViT/ResNet) and N captions into textual embeddings using a standard text transformer. Then, it attempts to learn a shared multi-modal embedding space by first linearly projecting each of the visual and textual embeddings into the shared embedding space and then maximising the cosine similarity between the correct N (image,text) embedding pairs and minimizing the similarity between the rest (N*N - N) embedding pairs. 

But, for computing the similarity, we require a single aggregated embedding for each image and each caption. This is obtained by taking the [CLS] embedding of the last layer of the ViT for the image and the [EOS] embedding of the last layer of the text transformer for the caption.   

One nuance is that the paper uses a symmetric cross entropy loss function which accounts for matching every image with its associated text and every text with its associated image. 

$$\mathcal{L} = \frac{1}{2}(\mathcal{L}_{I \to T} + \mathcal{L}_{T \to I})$$

where $$\mathcal{L}_{I \to T}$$ and $$\mathcal{L}_{T \to I}$$ are as follows:

$$\mathcal{L}_{I \to T} = -\frac{1}{N} \sum_{i=1}^{N} \log \frac{\exp(\mathbf{I}_i \cdot \mathbf{T}_i / \tau)}{\sum_{j=1}^{N} \exp(\mathbf{I}_i \cdot \mathbf{T}_j / \tau)}$$

$$\mathcal{L}_{T \to I} = -\frac{1}{N} \sum_{i=1}^{N} \log \frac{\exp(\mathbf{T}_i \cdot \mathbf{I}_i / \tau)}{\sum_{j=1}^{N} \exp(\mathbf{T}_i \cdot \mathbf{I}_j / \tau)}$$

<figure style="max-width: 520px; margin: 1.5rem auto;">
  <img src="/assets/img/blog/CLIP.png" alt="CLIP contrastive pre-training" style="width: 100%;">
  <figcaption class="caption">Figure 1: CLIP's contrastive pre-training allows zero-shot transfer to unseen object categories at test-time.</figcaption>
</figure>

#### 1.2 Zero-shot Task Transfer

The dataset used to train CLIP comprised of 400 million (image,text) pairs scraped from publically available sources on the internet. It is therefore safe to infer that the zero-shot transfer here means that the model is uitilized for a different task than the one it was  pre-trained on. Specifically, the pre-training task comprised of associating images and captions correctly with each other while at test-time the model can be used for object classification.

The model computes the feature embeddings of the image and a set of possible captions/texts, calculates the cosine-similarity between the set of all possible pairs and the (image,text) with the highest similarity is predicted as the most probable pair.

<hr style="margin-top: 2.5rem; margin-bottom: 2.5rem; border: none; border-top: 1px solid var(--global-text-color-light);">

### 2. Flamingo: a Visual Language model for Few Shot Learning (Alayrac et al. - Nov 2022)

Flamingo attempts to leverage the power of pre-trained visual and language models, handle sequences of arbitrarily interleved image and textual data and seamlessly integrate images and video inputs.

Taking inspiration from the few-shot learning capabilities of GPT-3, the authors propose a novel architecture that achieves state-of-the-art results on a wide array of image and video related tasks; outperforming the current best methods which involve fine-tuning on thousands of annotated examples.

<figure style="max-width: 520px; margin: 1.5rem auto;">
  <img src="/assets/img/blog/Flamingo.png" alt="Flamingo architecture" style="width: 100%;">
  <figcaption class="caption">Figure 2: Flamingo architecture overview</figcaption>
</figure>

#### 2.1 Gated cross-attention layers

The gated cross-attention layers are interleved with pre-initialized LM layers which provides an expressive way for autoregressive text generation conditioned on visual features. To ensure training stability, the architecture also incorporates tanh-gating (initialized at 0) so that the results of the initial forward pass match the results of the pretrained LM.

The keys and values in these layers are obtained from the visual features and the queries are derived from language inputs.

<figure style="max-width: 520px; margin: 1.5rem auto;">
  <img src="/assets/img/blog/xattn.png" alt="Flamingo cross-attention" style="width: 100%;">
  <figcaption class="caption">Figure 3: Gated Cross-attention layers interleved with LM layers to incorporate autogressive text generation conditioned on visual features</figcaption>
</figure>

#### 2.2 Perceiver Resampler

The Perceiver Resampler reduces the complexity of vision-text cross attention by using a predefined number of latent vectors which learn to aggregate/extract visual information during training. The visual features obtained from the visual encoder are flattened and converted into keys and values. These are further concatenated with keys and values derived from the learned latent vectors. The queries are derived from the latent vectors. These are then passed into attention layers and the final output tokens (same in number to the latent vectors) are sent to the cross-attention layers.
This results in a system indepedent of image resolution and no. of video frames.

<figure style="max-width: 520px; margin: 1.5rem auto;">
  <img src="/assets/img/blog/Perceiver_Resampler.png" alt="Perceiver Resampler" style="width: 100%;">
  <figcaption class="caption">Figure 4: The perceiver resampler maps a variable number of visual features into a fixed number of output tokens reducing complexity of vision-text cross-attention</figcaption>
</figure>

#### 2.3 Noteworthy Ablations

1) Ablations studies indicate that freezing the LM layers prevents catastrophic forgetting and is computationally cheaper as there are no gradient updates from training on an additional dataset.

2) 3 dataset mixing strategies were studied namely - data merged, round-robin and accumulation. Data merged involved constructing batches with a mixture of data from each dataset and round-robin entailed gradient updates seperately from batches of each dataset. However, the strategy of computing gradients from a batch of each dataset and using their weighted sum to update the parameters outperformed the previous methods.

3) For a textual token, how many previous images to cross-attend on was another ablation and the authors found that attending on the latest previous image yielded better results. An explaination offered is that there is no explicit way of disambiguating between different images.
They explored some ways of alleviate this by modifying image tags to include an index (image 1, image 2) or learning index embeddings but these were not robust enough to survive variation in image count during training and testing. 

<hr style="margin-top: 2.5rem; margin-bottom: 2.5rem; border: none; border-top: 1px solid var(--global-text-color-light);">

### 3. BLIP-2: Bootstrapping Language-Image Pretraining with Frozen Image Encoders and Language Models (Li et al. - Jun 2023)

BLIP-2 positions itself as an effective and efficient way to leverage the capabilities of Frozen Image encoders and LLMs instead of end-to-end pretraining which is more compute and memory intensive. The main problem is to ensure cross-modal alignment for which they propose a Querying Transformer.

#### 3.1 Q-Former Architecture

Q-former employes learnable query vectors, acting a information bottleneck and feeding only the most relevant visual features to the LLM. It comprises of 2 transformer submodules that share the same self-attention layers. One of them is an vision transformer which interacts with the frozen image encoder for visual feature extraction. A text transformer is the other submodule.

It utilizes a 2-stage pretraining framework where the first stage focuses on visual-language representation learning, allowing the model to learn visual features most relevant to the text. The second stage focuses on making the output visual representations of the Q-former interpretable for the LLM.

<figure style="max-width: 520px; margin: 1.5rem auto;">
  <img src="/assets/img/blog/Q-Former.png" alt="Q-Former" style="width: 100%;">
  <figcaption class="caption">Figure 5: Objectives for stage 1 of pretraining and architecture of the Querying Transformer</figcaption>
</figure>

#### 3.2 Stage I: Vision-Language Representation Learning 

This stage involves optimizing 3 objectives to ensure that the Q-former learns visual information most relevant to the text. 

1) Image-Text Contrastive Learning (ITC) - This task entails the same contrastive learning optimization as in CLIP by maximising the similarity between the output query embeddings from Q-former and the [CLS] text embeddings from the BERT-like text transformer. Since the Q-former outputs contains multiple query embeddings, the papers computes the similarity with each query and then selects the highest one to optimize. A unimodal attention mask is utilized to facilitate this task.

2) Image-Grounded Text Generative (ITG) - This task trains the model to generate texts given the input images as conditionals. For this, all of the information present in the image needs to be extracted by the query embeddings. So, the queries can attend to each other but not with the text embeddings while the text embeddings can interact with previous texts and with all the queries. This can be ensured with a multi-modal causal attention mask.

3) Image-Text Matching (ITM) - This tasks attempts to classify whether each image-text pair is positive (matching) or negative (unmatching) allowing the model to learn fine-grained alignment.
A bi-directional attention mask is utilized allowing all pair query-text interaction. A set of logits is derived from the output query embeddings using a linear classifier which are then averged to produce an output-matching score.

<figure style="max-width: 520px; margin: 1.5rem auto;">
  <img src="/assets/img/blog/Masks.png" alt="Masks" style="width: 100%;">
  <figcaption class="caption">Figure 6: Attention Masks for each objective</figcaption>
</figure>

#### 3.3 Stage II: Vision-to-Language Generative Learning

The second stage involves training a linear projector that sends the Q-formers learned output vectors as soft visual inputs to the LLM. Since the Q-former is extensively trained to learn cross-modal representations in stage I, there is less burden on the LLM to learn cross-modal alignment now. 

The paper experiments with 2 kinds of LLMs:- encoder-decoder and decoder-only. The decoder-only version uses language modelling loss to generate visual conditioned text. The encoder-decoder uses a prefix language modelling loss, where the the target is suffix prediction using a prefix text and the visual queues. 

<figure style="max-width: 520px; margin: 1.5rem auto;">
  <img src="/assets/img/blog/Stg2.png" alt="Q-Former" style="width: 100%;">
  <figcaption class="caption">Figure 7: Stage 2 of the pretraining focuses on cross-modal alignment</figcaption>
</figure>

<hr style="margin-top: 2.5rem; margin-bottom: 2.5rem; border: none; border-top: 1px solid var(--global-text-color-light);">