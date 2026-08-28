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

### Learning Transferable Visual Models from Natural Language Supervision (Radford et al. - Feb 2021)

This paper attempts to address the prevalant limitation of computer systems at that time to predict only a fixed set of categories on which system is trained. The authors say that this limits the generalizability and usability of such systems and learning directly from text associated with images is promising research direction to alleviate this limitation. So, they demonstrate that a simple pre-training task of associating which (image,text) pairs go together is an efficient and scalable way to learn image representations which can enable zero-shot downstream tasks. 

#### Contrastive Pre-training Objective

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

#### Zero-shot Task Transfer

The dataset used to train CLIP comprised of 400 million (image,text) pairs scraped from publically available sources on the internet. It is therefore safe to infer that the zero-shot transfer here means that the model is uitilized for a different task than the one it was  pre-trained on. Specifically, the pre-training task comprised of associating images and captions correctly with each other while at test-time the model can be used for object classification.

The model first computes the feature embeddings of the image and a set of possible captions/texts, calculates the cosine-similarity of the set of all possible pairs and the image,caption/text with the highest similarity is predicted as the most probable pair.

<br>
