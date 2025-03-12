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

The figure below shows ALPACA UI written using GRADIO

![ALPACA UI](https://github.com/rFronteddu/rfronteddu.github.io/blob/main/img/alpaca_ui.png?raw=true)


### ALPACA System Design

![ALPACA System Design](https://github.com/rFronteddu/rfronteddu.github.io/blob/main/img/alpaca_arch_1.png?raw=true)

The figure above shows ALPACA architecture.

#### System Requirements
The first prototype was a simple single-page application built with Gradio. However, it had limitations: once the page was closed, the application state was lost, and job scheduling wasn’t possible. Additionally, the app lacked security.

The main objectives for the new system were:

* Support 10-50 concurrent users
* Implement security
* Add job scheduling functionality

#### High-level Design Discussion
To address the limitations of the prototype, we divided the system into three Python components:

1. **UI**: The user interface for interacting with ALPACA (based on GRADIO).
2. **Backend**: A service to preserve state and provide a REST API (powered by FastAPI).
3. **Task Scheduler**: Responsible for managing LLM interactions.

We used OLLAMA for LLM functionalities.

When a user submits a job, an entry is created in the backend and stored in a database. The UI can query the task status for the logged-in user, allowing them to view, cancel, or retrieve results.

The Task Manager monitors job statuses via the backend's REST API, handling job execution and updating the status with results or errors.

For security, we deployed a Keycloak instance for OAuth authentication. Each component was assigned a unique client ID, ensuring that services could only perform specific actions (e.g., UI can read, cancel, and add tasks, while the Task Scheduler can update status).

Additionally, to further enhance security, we encrypted all data stored on disk separately for each user. We used a hash of a secret the user provides to create a unique encryption key, reducing the chances of data leaks.

To secure the Gradio frontend, we split it into two apps: one for the login (unauthenticated) and another for the secured UI.

All services communicate with Keycloak to validate user access and ensure proper service authentication. Since the system is deployed on-premise, we generated self-signed certificates for secure HTTPS connections between services.

Since OLLAMA doesn’t support HTTPS, we used NGINX as a reverse proxy. This setup also allowed us to generalize GPU access by referring to GPUs as GPU_RESOURCE_IP.gpu_1, gpu_2, etc.



