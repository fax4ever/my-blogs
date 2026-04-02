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

## Models Comparison

|  | Ambient patient blueprint | IT self-service agent  |
| :---- | :---- | :---- |
| REASONING MODELS (70B Parameters) | Llama-3.3-70B-Instruct | Llama-3-70B-Instruct w8a8 |
| SAFETY MODELS (8B Parameters) | NemoGuard-8B | PromptGuard, Llama Guard 3 |
| SPEECH MODELS (1B-3B Parameters) | Parakeet-CTC-1.1B, Magpie Multilingual | \- |
| EMBEDDING MODELS (110M-400M Parameters) | \- | all-MiniLM-L6-v2 |
| EVALUATION MODELS | \- | Llama 4 scout 16B w4a16 |

# UI Channels and User Experience

# CI/CD and Evaluation pipelines

# Tracing and Observability

