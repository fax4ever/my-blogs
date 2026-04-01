# Agentic Applications Comparisons

This document evaluates the differences and similarities between two agentic applications:

**Red Hat IT Self-Service Agent Quickstart**
[https://github.com/rh-ai-quickstart/it-self-service-agent](https://github.com/rh-ai-quickstart/it-self-service-agent)  
A quickstart agentic application developed by the Red Hat Ecosystem AI team.

**NVIDIA Ambient Patient Blueprint**
[https://github.com/NVIDIA-AI-Blueprints/ambient-patient](https://github.com/NVIDIA-AI-Blueprints/ambient-patient)  
An NVIDIA blueprint agentic application for healthcare scenarios.

# Agentic Orchestration

Both applications use [LangGraph](https://www.langchain.com/langgraph), which is a popular orchestration framework used to define agentic workflows.
Both use the [routing agentic pattern](https://www.theagenticwiki.com/docs/patterns/routing/), starting from a routing or primary assistant agent that delegates the rest of the conversation to specialized agents.    
The IT self-service agent currently has only one specialization: Laptop Refresh Agent.

| Specialized Agent | Purpose |
| :---- | :---- |
| Laptop Refresh Agent | Validates eligibility, gathers requirements, creates ticket for laptop refresh |
| \[Extensible\] | Framework supports multiple IT process agents |

The Ambient Patient blueprint defines three specialized agents:

| Specialized Agent | Purpose |
| :---- | :---- |
| Patient Intake Specialist | Collects demographics, symptoms, medications, allergies |
| Appointment Making Specialist | Checks for availability, books appointments |
| Medication Lookup Specialist | Fast Healthcare Interoperability Resources (HL7 standard for exchanging healthcare information electronically) queries |

Specialized agents have isolated tool sets, making it possible to evolve each independently, improving cohesion and reducing coupling between components.

## State machine management

LangGraph is fundamentally a state machine framework for building agentic workflows. State flows through the graph. Each node reads state, executes logic, and returns state updates.

The NVIDIA Ambient Patient blueprint does not persist state, meaning that if the service restarts, all conversation state is lost. Additionally, conversations cannot be distributed across different pods. This configuration is suitable for development, testing, or demonstration purposes, but is not generally appropriate for production environments.

The design uses one thread per conversation (identified by patient-id), which guarantees isolation but limits flexibility and distribution possibilities.

In contrast, the IT Self-Service agent follows a session-based conversation design that is more complex but allows conversations to be persisted and recovered. This mechanism enables users to start a conversation using one channel (e.g., Slack) and continue it using a different channel (e.g., corporate email).

The conversation state is fully serialized and persisted to a PostgreSQL database. This design enables service restarts, multi-replica deployments, and concurrent conversations, providing production-level reliability, particularly when deployed on a Kubernetes cluster where pods can be stopped and restarted on any node at any time.

# Protocols Comparison

# Models Comparison

# UI Channels and User Experience

# CI/CD and Evaluation pipelines

# Tracing and Observability

