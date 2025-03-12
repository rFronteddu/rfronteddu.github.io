---
layout: post
title:  "ALPACA System Design Notes"
date:   2025-03-12 11:39:23 -0500
categories: blog
---

## Introduction

Welcome to my first blog post! This is an example where I will walk you through adding figures and describing what's going on.

### What is ALPACA

ALPACA is a tool I was tasked with designing to leverage machine learning (ML) for accelerating the process of tracking and evaluating projects awarded by armed forces organizations in alignment with Department of Defense (DoD) priorities.

The project began as a simple prototype, where users could upload a CSV file containing project details. The tool would then analyze these projects using a large language model (LLM) against a set of predefined metrics, guided by a specialized prompt.

Our team consisted of me as the tech lead, my supervisor as the subject matter expert, and a machine learning engineer who focused on prompt refining and continuous model evaluation. After securing initial funding, the project scope expanded to include more robust features, such as job scheduling (to handle long processing times) and enhanced security measures to meet higher operational standards.

### ALPACA System Design

![ALPACA System Design](https://github.com/rFronteddu/rfronteddu.github.io/blob/main/img/alpaca_arch_1.png)

The figure above shows ALPACA architecture.


