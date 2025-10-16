---
title: "Dynamic GPU Slicing with Red Hat OpenShift and NVIDIA MIG"
description: "On-demand GPU resource allocation for flexible AI/ML workloads"
date: 2025-10-14
---

*Author: [Harshal Patil](https://github.com/harche){:target="_blank" rel="noopener"}*

*This article was originally published on [Red Hat Developer](https://developers.redhat.com/articles/2025/10/14/dynamic-gpu-slicing-red-hat-openshift-and-nvidia-mig){:target="_blank" rel="noopener"}.*

---

NVIDIA Multi-Instance GPU (MIG) allows dynamic GPU resource allocation in Kubernetes, enabling more efficient and flexible GPU usage across different AI/ML workloads. The dynamic accelerator slicer operator in Red Hat OpenShift brings this power to developers in an on-demand, developer-friendly way.

## The Challenge

Running multiple AI models with vastly different resource needs on the same GPU infrastructure creates a dilemma: do you provision for the largest model and waste resources when running smaller ones, or do you manage separate GPU pools for different model sizes? Neither approach is ideal.

## The Solution

MIG turns one GPU into many independent slices, each with isolated memory and compute resources. Combined with the dynamic accelerator slicer operator in OpenShift, you get:

- **Flexible allocation**: Create GPU slices on-demand based on workload requirements
- **Better utilization**: Run seven tiny models, two medium models, or one large model on the same physical GPU
- **Kubernetes-native**: Managed through standard Kubernetes resources and workflows

## Real-World Examples

The article demonstrates three GPU slicing scenarios using vLLM:

1. **Seven tiny models** (Gemma 3 270M) on 1g.5gb slices
2. **Two medium models** (Qwen2-7B-Instruct) on 3g.20gb slices
3. **One large model** (GPT-OSS 20B) on a 7g.40gb slice

Each scenario shows how the same physical GPU can be dynamically reconfigured to match the exact needs of different AI workloads, eliminating resource waste and improving overall cluster utilization.

---

**Read the full article on Red Hat Developer**: [Dynamic GPU Slicing with Red Hat OpenShift and NVIDIA MIG](https://developers.redhat.com/articles/2025/10/14/dynamic-gpu-slicing-red-hat-openshift-and-nvidia-mig){:target="_blank" rel="noopener"}
