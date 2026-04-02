# Agentic Applications Comparison

This document evaluates the differences and similarities between two agentic applications:

**Red Hat IT Self-Service Agent Quickstart**  
[https://github.com/rh-ai-quickstart/it-self-service-agent](https://github.com/rh-ai-quickstart/it-self-service-agent)  
A quickstart agentic application developed by the Red Hat Ecosystem AI team.

**NVIDIA Ambient Patient Blueprint**  
[https://github.com/NVIDIA-AI-Blueprints/ambient-patient](https://github.com/NVIDIA-AI-Blueprints/ambient-patient)  
An NVIDIA blueprint agentic application for healthcare scenarios.

# Agentic Workflow / Orchestration

Both applications use [LangGraph](https://www.langchain.com/langgraph), which is a popular orchestration framework used to define agentic workflows. 

## Routing to Specialist Agent

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

The routing in the Ambient Patient blueprint is implemented using a keyword-based approach:

```` ```python ````  
`def route_to_assistant(state: State) -> str:`  
    `"""Route based on message content analysis"""`  
      
    `last_message = state["messages"][-1]`  
    `content = last_message.content.lower()`  
      
    `# Keyword matching`  
    `if "appointment" in content or "schedule" in content:`  
        `return "appointment_assistant"`  
    `elif "medication" in content or "prescription" in content:`  
        `return "medication_assistant"`  
    `elif "intake" in content or "check in" in content:`  
        `return "patient_intake_assistant"`  
    `else:`  
        `return "primary_assistant"`

`# Graph configuration`  
`graph.add_conditional_edges(`  
    `source="primary_assistant",`  
    `path=route_to_assistant,`  
    `path_map={`  
        `"appointment_assistant": "appointment_assistant",`  
        `"medication_assistant": "medication_assistant",`  
        `"patient_intake_assistant": "patient_intake_assistant",`  
        `"primary_assistant": "primary_assistant"`  
    `}`  
`)`  
```` ``` ````

This is not as flexible and powerful as the LLM-based intent classification implemented by the IT self-service agent:

```` ```python ````  
`def route_request(state: ConversationState) -> str:`  
    `"""Route based on LLM intent classification"""`  
      
    `last_message = state["messages"][-1]`  
      
    `# LLM classifies intent`  
    `classification_prompt = f"""`  
    `Classify the user's intent from the following categories:`  
    `- laptop_refresh: User wants new laptop`  
    `- password_reset: User needs password help`  
    `- access_request: User needs system access`  
    `- general: Other IT questions`  
      
    `User message: {last_message.content}`  
      
    `Respond with only the category name.`  
    `"""`  
      
    `intent = llm.invoke(classification_prompt).content.strip()`  
      
    `# Map intent to specialist`  
    `intent_map = {`  
        `"laptop_refresh": "laptop_refresh_agent",`  
        `"password_reset": "password_reset_agent",`  
        `"access_request": "access_request_agent",`  
        `"general": "general_routing_agent"`  
    `}`  
      
    `return intent_map.get(intent, "general_routing_agent")`

`# Graph configuration`  
`graph.add_conditional_edges(`  
    `source="routing_agent",`  
    `path=route_request`  
`)`  
```` ``` ````

## Inter-Agent Communication

Agents don't call each other directly. They communicate via the **shared state** managed by LangGraph.

```` ``` ````  
`1. User Input → Graph`  
   `Graph State = {messages: [HumanMessage("I need a laptop")]}`

`2. Graph invokes: router_agent(state)`  
   `router_agent:`  
     `- reads: state["messages"]`  
     `- processes: intent classification`  
     `- returns: {intent: "laptop_refresh", messages: [AIMessage("routing...")]}`

`3. Graph merges update into state`  
   `Graph State = {`  
     `messages: [HumanMessage(...), AIMessage("routing...")],`  
     `intent: "laptop_refresh"`  
   `}`

`4. Graph evaluates routing condition`  
   `route_next(state) → "laptop_refresh_agent"`

`5. Graph invokes: laptop_refresh_agent(state)`  
   `laptop_refresh_agent:`  
     `- reads: state["messages"], state["intent"]`  
     `- processes: eligibility check`  
     `- returns: {eligibility: "approved", messages: [AIMessage("You're eligible!")]}`

`6. Graph merges update`  
   `Graph State = {`  
     `messages: [..., AIMessage("You're eligible!")],`  
     `intent: "laptop_refresh",`  
     `eligibility: "approved"`  
   `}`  
```` ``` ````

This strategy is applied consistently by both applications. The communication is synchronous and not direct agent-to-agent, but mediated by the state. Also note that the state is not modified directly by the agent, limiting the possibility for an agent to modify fields outside its area of competency.

## State machine management

LangGraph is fundamentally a state machine framework for building agentic workflows. State flows through the graph. Each node reads state, executes logic, and returns state updates.

The NVIDIA Ambient Patient blueprint does not persist state, meaning that if the service restarts, all conversation state is lost. Additionally, conversations cannot be distributed across different pods. This configuration is suitable for development, testing, or demonstration purposes, but is not generally appropriate for production environments.

The design uses one thread per conversation (identified by patient-id), which guarantees isolation but limits flexibility and distribution possibilities.

In contrast, the IT Self-Service agent follows a session-based conversation design that is more complex but allows conversations to be persisted and recovered. This mechanism enables users to start a conversation using one channel (e.g., Slack) and continue it using a different channel (e.g., corporate email).

The conversation state is fully serialized and persisted to a PostgreSQL database. This design enables service restarts, multi-replica deployments, and concurrent conversations, providing production-level reliability, particularly when deployed on a Kubernetes cluster where pods can be stopped and restarted on any node at any time.

## Agent-to-Tool Protocols

The Ambient Patient blueprint defines agents and LLM tools directly at the LangGraph level, following the LangChain syntax and binding them using LangGraph tool nodes that link them to the agent nodes:

**Tool Definition:**  
```` ```python ````  
`from langchain_core.tools import tool`

`@tool`  
`def get_patient_medications(patient_id: str) -> str:`  
    `"""Fetch patient medications from FHIR server"""`  
      
    `from fhirclient import client`  
      
    `# Direct API call in same process`  
    `fhir_client = client.FHIRClient(settings={`  
        `'app_id': 'ambient_patient',`  
        `'api_base': 'https://r4.smarthealthit.org'`  
    `})`  
      
    `# Query FHIR`  
    `meds = fhir_client.server.request_json(`  
        `f'MedicationRequest?patient={patient_id}'`  
    `)`  
      
    `# Process and return`  
    `med_list = [med['medicationCodeableConcept']['text'] for med in meds['entry']]`  
    `return f"Patient medications: {', '.join(med_list)}"`  
```` ``` ````

**Agent-Side Usage:**  
```` ```python ````  
`from langgraph.prebuilt import ToolNode`

`# Tools registered with agent`  
`tools = [get_patient_medications, book_appointment, search_medication_info]`

`# Create tool node`  
`tool_node = ToolNode(tools)`

`# Graph includes tool node`  
`graph.add_node("tools", tool_node)`  
`graph.add_edge("medication_assistant", "tools")`  
```` ``` ````

While the IT self-service agent uses LangGraph agents and tools defined with [Llama Stack](https://www.llama.com/products/llama-stack/), where tools are defined to follow the [Model Context Protocol](https://en.wikipedia.org/wiki/Model_Context_Protocol).

```` ```yaml ````  
`name: "laptop-refresh"`  
`...`  
`lg_state_machine_config: "config/lg-prompts/lg-prompt-big.yaml"`  
`...`  
`sampling_params:`  
  `strategy:`  
    `type: "top_p"`  
    `temperature: 0.7`  
    `top_p: 0.95`  
`mcp_servers:`  
  `- name: "snow"`  
    `uri: "http://mcp-self-service-agent-snow:8000/mcp"`  
    `require_approval: "never"`  
`knowledge_bases: ["laptop-refresh"]`  
```` ``` ````

This approach is more complex; it requires, for instance, deploying a workload for each tool that needs to be called. On the other hand, tools can be reused across different applications and agents in a very flexible way. Moreover, the MCP protocol allows decoupling of the agent from the tools, making it impossible for the tool process to access the agent's state memory, guaranteeing security boundaries — credentials never leave the MCP servers, for instance.

## Agent-to-LLM Protocols

Both applications use OpenAI-compatible protocols. In particular, the Ambient Patient blueprint uses [NVIDIA NIM](https://developer.nvidia.com/nim?sortBy=developer_learning_library%2Fsort%2Ffeatured_in.nim%3Adesc%2Ctitle%3Aasc) APIs (which are OpenAI-compatible):

```` ```json ````  
`{`  
  `"model": "meta/llama-3.3-70b-instruct",`  
  `"messages": [`  
    `{`  
      `"role": "system",`  
      `"content": "You are a helpful medical assistant."`  
    `},`  
    `{`  
      `"role": "user",`  
      `"content": "What medications am I on?"`  
    `}`  
  `],`  
  `"tools": [`  
    `{`  
      `"type": "function",`  
      `"function": {`  
        `"name": "get_patient_medications",`  
        `"description": "Fetch patient medications from FHIR",`  
        `"parameters": {`  
          `"type": "object",`  
          `"properties": {`  
            `"patient_id": {"type": "string"}`  
          `},`  
          `"required": ["patient_id"]`  
        `}`  
      `}`  
    `}`  
  `],`  
  `"tool_choice": "auto",`  
  `"temperature": 0.7,`  
  `"max_tokens": 1024,`  
  `"stream": true`  
`}`  
```` ``` ````

While the IT self-service agent quickstart uses the OpenAI-compatible API exposed by Llama Stack:

```` ```json ````  
`{`  
  `"model": "meta-llama/Llama-3-70b-chat-hf",`  
  `"messages": [`  
    `{`  
      `"role": "system",`  
      `"content": "You are an IT support assistant."`  
    `},`  
    `{`  
      `"role": "user",`  
      `"content": "I need a new laptop"`  
    `}`  
  `],`  
  `"tools": [`  
    `{`  
      `"tool_name": "get_employee_by_email",`  
      `"description": "Fetch employee from ServiceNow",`  
      `"parameters": {...}`  
    `}`  
  `],`  
  `"stream": false,`  
  `"temperature": 0.7,`  
  `"max_tokens": 1024`  
`}`  
```` ``` ````

## RAG and Knowledge Base Integration

The IT self-service agent integrates a RAG-based knowledge base, exposed as an MCP server, which allows the Laptop Refresh Agent to query policy documents — for example, to determine laptop refresh eligibility rules. This grounds the agent's decisions in up-to-date enterprise knowledge rather than relying solely on what was encoded in the model at training time.

The Ambient Patient blueprint does not include a RAG component. The Medication Lookup agent queries a [FHIR](https://hl7.org/fhir/) server directly for patient-specific data and uses [Tavily](https://tavily.com/) for general medication information searches, but there is no mechanism for grounding responses in a curated internal knowledge base.

## Models Comparison

|  | Ambient patient blueprint | IT self-service agent  |
| :---- | :---- | :---- |
| REASONING MODELS (70B Parameters) | Llama-3.3-70B-Instruct | Llama-3-70B-Instruct w8a8 |
| SAFETY MODELS (8B Parameters) | NemoGuard-8B | PromptGuard, Llama Guard 3 |
| SPEECH MODELS (1B-3B Parameters) | Parakeet-CTC-1.1B, Magpie Multilingual | \- |
| EMBEDDING MODELS (110M-400M Parameters) | \- | all-MiniLM-L6-v2 |
| EVALUATION MODELS | \- | Llama 4 scout 16B w4a16 |

## Safety & Guardrails

Both applications treat guardrails as optional components that can be enabled or disabled via configuration, rather than hard-wiring them into the agents.

The Ambient Patient blueprint integrates [NeMo Guardrails](https://github.com/NVIDIA/NeMo-Guardrails), configured through YAML-based Colang flows. This allows fine-grained customisation of which topics the model may and may not discuss, as well as response shaping. Safety is enforced by the NemoGuard-8B Content Safety and Topic Control models.

The IT self-service agent takes a different approach, combining [PromptGuard](https://www.llama.com/docs/model-cards-and-prompt-formats/prompt-guard) for prompt injection detection with [Llama Guard 3](https://www.llama.com/docs/model-cards-and-prompt-formats/llama-guard-3) for content safety. While less customisable in terms of topic control, this approach is more focused on preventing adversarial inputs from reaching the model.

The key philosophical difference is that NVIDIA focuses on *what the model is allowed to say*, while Red Hat focuses on *preventing malicious inputs from reaching the model in the first place*.

|  | Ambient patient blueprint | IT self-service agent |
| :---- | :---- | :---- |
| GUARDRAILS FRAMEWORK | NeMo Guardrails 0.17.0 | PromptGuard \+ Llama Guard 3 |
| CONTENT SAFETY MODEL (8B) | NemoGuard-8B Content Safety | Llama Guard 3 |
| TOPIC CONTROL | NemoGuard-8B Topic Control | \- |
| PROMPT INJECTION PROTECTION | \- | PromptGuard |
| CONFIGURATION | YAML-based Colang flows | Service-based integration |
| DEPLOYMENT | Optional, configurable via env var | Optional service (disabled by default) |

# Other Aspects

## Deployment Environment

The Ambient Patient blueprint is designed to be deployed using Docker Compose, which is primarily designed for managing multi-container applications on a single host.  
It is also quite difficult to configure for end-to-end operation when deployed on a Kubernetes cluster. For instance, [WebRTC](https://webrtc.org/) communication between the browser and the Kubernetes cluster requires a TURN server to support NAT traversal and relay. This approach was adopted, for instance, in [our fork of the Ambient Patient blueprint](https://github.com/RHEcosystemAppEng/ambient-patient), in which we extended support for deployment on an [OpenShift](https://www.redhat.com/en/technologies/cloud-computing/openshift) cluster.  
On the other hand, the IT self-service agent was designed as a Kubernetes/OpenShift-native application. Everything is designed to be straightforward to deploy on container platforms.

The two applications also follow different scalability models. The Ambient Patient blueprint scales *vertically* — more GPUs are added to host larger or additional NIM inference endpoints. The IT self-service agent scales *horizontally* — its services are stateless pods that can be replicated freely, with state externalised to PostgreSQL and message routing decoupled via Kafka and Knative, so any pod can handle any request at any time.

## Event-driven Architecture

The IT self-service agent is built on an event-driven architecture using [Apache Kafka](https://kafka.apache.org) and [Knative Eventing](https://knative.dev/docs/eventing/) (CloudEvents). When a user sends a message via Slack or email, the Integration Dispatcher publishes it as a Kafka event on a request topic. The Request Manager consumes the event asynchronously, invokes the LangGraph agent, and publishes the response back to a response topic. The Integration Dispatcher then delivers it to the appropriate channel.

This design decouples front-end channels from agent processing, enabling the system to absorb high message volumes without blocking, survive transient failures, and scale individual components independently.

The Ambient Patient blueprint is fully synchronous by contrast: a user message is handled in a single request-response cycle with token streaming. This is well suited for real-time voice interaction but limits the system's ability to scale under concurrent load or to recover gracefully from service interruptions.  

## UI Channels and User Experience

The Ambient Patient blueprint delivers a rich and immersive user experience, making extensive use of full-duplex protocols — WebSockets for text and WebRTC for audio. It provides two interfaces: a text-based one and a voice-based one that also plays responses as audio. The overall user experience is excellent.  
The IT self-service agent does not expose a web interface, but it allows users to interact through different channels: Slack and email. This makes the application better suited for production environments, though potentially less suitable as a demo, since configuring an SMTP/IMAP email server or a Slack workspace takes some time.

## Real-time Audio Pipeline

The voice interface of the Ambient Patient blueprint is backed by a multi-layer real-time audio pipeline, which has no equivalent in the IT self-service agent:

1\. The browser captures microphone audio and establishes a [WebRTC](https://webrtc.org/) connection (Opus codec, 48kHz) to a [Pipecat](https://www.pipecat.ai) Python backend, using STUN/TURN for NAT traversal.  
2\. Pipecat resamples the audio to 16kHz PCM and applies Voice Activity Detection (VAD) to detect speech boundaries.  
3\. The audio stream is forwarded to [NVIDIA RIVA](https://developer.nvidia.com/riva) ASR via \*\*gRPC\*\* (HTTP/2 \+ Protocol Buffers), which returns both interim and final transcripts in real time.  
4\. The final transcript is passed to the LangGraph agent, which produces a text response.  
5\. The response text is sent to RIVA TTS via gRPC, which streams back synthesised audio chunks.  
6\. Pipecat relays the audio back to the browser over the WebRTC connection.

The use of gRPC for the RIVA services is deliberate: its bidirectional streaming over HTTP/2 is significantly more efficient than REST for continuous audio data, contributing to the sub-100ms end-to-end latency of the pipeline.

## CI/CD and Evaluation pipelines

The Ambient Patient blueprint defines no CI/CD pipelines in its codebase, while the IT self-service agent provides GitOps-based pipelines to enable automatic testing on incoming pull requests and before new releases.  
The IT self-service agent also incorporates a vLLM as a judge in the test loop, using [DeepEval](https://deepeval.com/).

## Tracing and Observability

The IT self-service agent provides enterprise-grade observability: OpenTelemetry integration, distributed tracing across all workloads, structured log management, component-specific performance metrics, and comprehensive evaluation frameworks. The Ambient Patient blueprint, by contrast, limits tracing to agent-level visibility via LangSmith, provides no further tracing integration, and relies on Docker stdout for logs.

## Python Project Management

The Ambient Patient blueprint manages dependencies with pip and a requirements.txt file, while the IT self-service agent makes extensive use of uv with pyproject.toml, enabling a more enterprise-grade approach to dependency management.  
