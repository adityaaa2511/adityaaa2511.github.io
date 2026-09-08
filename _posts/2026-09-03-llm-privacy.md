---
layout: post
title: "Can LLMs leak your private data?"
date: 2026-09-03 21:00:00 +0530
description: A comprehensive analysis of LLM privacy papers
tags: [AI safety, LLM privacy]
comments: false
_styles: >
  .post-title { font-weight: 500; margin-bottom: 0.75rem; }
  .post-meta { margin-top: 0.6rem; margin-bottom: 0.5rem; }
  .post-content h3, .post-content h4 { font-weight: 500; margin-top: 2rem; margin-bottom: 0.75rem; }
  .post-content p { line-height: 1.75; margin-bottom: 1.25rem; }
---

### Background

AI safety is a hot topic and this post aims to dissect some of the papers related to AI safety and LLM privacy as part of a comprehensive effort to understand the technicalities of this field. Recent research shows that adversaries can extract training data from LLM which can leak personally identifiable information (PII) which can result in catastrophic consequences. The post details how adversaries can maliciously attack LLMs for extracting sensitive data and some strategies some mitigating such attacks.



