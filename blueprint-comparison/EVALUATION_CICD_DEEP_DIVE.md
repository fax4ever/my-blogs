# Evaluation & CI/CD Deep Dive: NVIDIA vs Red Hat Agentic Architectures

**Purpose:** Comprehensive analysis of evaluation frameworks, testing strategies, GitOps pipelines, and CI/CD workflows in both architectures.

**Date:** April 1, 2026

---

## Table of Contents

1. [Evaluation Framework Overview](#1-evaluation-framework-overview)
2. [Red Hat DeepEval Framework](#2-red-hat-deepeval-framework)
3. [NVIDIA Evaluation Gap](#3-nvidia-evaluation-gap)
4. [Testing Strategies](#4-testing-strategies)
5. [GitOps & CI/CD Pipelines](#5-gitops--cicd-pipelines)
6. [Deployment Strategies](#6-deployment-strategies)
7. [Quality Gates & Release Criteria](#7-quality-gates--release-criteria)
8. [Monitoring & Production Validation](#8-monitoring--production-validation)
9. [Continuous Improvement Loop](#9-continuous-improvement-loop)
10. [Recommendations](#10-recommendations)

---

## 1. Evaluation Framework Overview

### 1.1 Why Evaluation Matters for Agents

**Traditional Software Testing:**
```
Unit Tests → Integration Tests → E2E Tests
- Deterministic: Same input → Same output
- Binary: Pass/Fail
- Fast feedback (<1 min)
```

**Agentic AI Testing:**
```
LLM Evaluation → Business Metrics → User Acceptance
- Non-deterministic: Same input → Different outputs
- Gradual: Quality scores (0-100)
- Slower feedback (minutes to hours)
```

**Key Challenges:**

| Challenge | Description | Impact |
|-----------|-------------|--------|
| **Non-determinism** | LLM responses vary with temperature | Hard to write traditional tests |
| **Subjective quality** | "Good" response is subjective | Need human evaluation or LLM-as-judge |
| **Multi-turn complexity** | Quality emerges over conversation | Single-turn tests insufficient |
| **Business logic** | "Did agent follow policy?" | Need domain-specific metrics |
| **Tool calling accuracy** | "Did agent call right tool with right args?" | Critical for agentic systems |

---

### 1.2 Evaluation Maturity Model

```
Level 0: No Evaluation
├─ Manual testing only
├─ No metrics tracked
└─ Hope for the best in production ❌

Level 1: Basic Metrics
├─ Track conversation length
├─ Monitor error rates
└─ User feedback surveys ⚠️

Level 2: Automated Evaluation (Red Hat is here)
├─ DeepEval framework
├─ Business-specific metrics
├─ Synthetic test generation
└─ CI/CD integration ✅

Level 3: Continuous Learning (Future)
├─ Online evaluation (A/B tests)
├─ Automatic prompt optimization
├─ Model fine-tuning loop
└─ Self-improving agents 🚀
```

**Current State:**
- **NVIDIA:** Level 1 (basic monitoring via LangSmith, no systematic evaluation)
- **Red Hat:** Level 2 (comprehensive DeepEval framework)

---

## 2. Red Hat DeepEval Framework

### 2.1 DeepEval Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    DeepEval Framework                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Test Conversations (Golden Dataset)                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ test_laptop_eligible.json                            │  │
│  │ test_laptop_not_eligible.json                        │  │
│  │ test_ambiguous_request.json                          │  │
│  │ test_multi_request.json                              │  │
│  └──────────────────────────────────────────────────────┘  │
│                         ↓                                    │
│  Agent Execution                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Run agent with test input                            │  │
│  │ Capture full conversation + state + tool calls       │  │
│  └──────────────────────────────────────────────────────┘  │
│                         ↓                                    │
│  Metric Evaluation (15+ metrics)                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Policy Compliance: Did agent enforce 3-year rule?    │  │
│  │ Info Gathering: Did agent collect all required data? │  │
│  │ Ticket Format: Is ticket number valid (RITM\d+)?     │  │
│  │ Tool Accuracy: Correct MCP calls with right args?    │  │
│  │ Conversation Quality: Helpful, concise, professional?│  │
│  └──────────────────────────────────────────────────────┘  │
│                         ↓                                    │
│  Results & Reporting                                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Pass/Fail per test                                   │  │
│  │ Aggregate scores                                     │  │
│  │ Regression detection                                 │  │
│  │ HTML/JSON reports                                    │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

### 2.2 DeepEval Metrics for Laptop Refresh Agent

**1. Policy Compliance Metric**

```python
from deepeval.metrics import GEval
from deepeval.test_case import LLMTestCase

# Define metric
policy_compliance = GEval(
    name="Policy Compliance",
    criteria="Agent correctly applies the 3-year laptop refresh policy",
    evaluation_params=[
        "Agent checks employee's laptop age",
        "Agent approves if age >= 3 years",
        "Agent rejects if age < 3 years",
        "Agent explains policy clearly"
    ],
    evaluation_steps=[
        "Verify laptop age was checked via ServiceNow MCP",
        "Verify eligibility decision matches policy",
        "Verify policy was communicated to user"
    ],
    threshold=0.8  # 80% score required to pass
)

# Test case
test_case = LLMTestCase(
    input="I need a new laptop",
    actual_output=agent_conversation,
    context={
        "employee_laptop_age": 4,
        "expected_outcome": "eligible"
    }
)

# Evaluate
score = policy_compliance.measure(test_case)
# score = 0.95 (PASS) - Agent correctly approved 4-year-old laptop
```

**2. Information Gathering Metric**

```python
from deepeval.metrics import GEval

info_gathering = GEval(
    name="Information Gathering",
    criteria="Agent collects all required information before creating ticket",
    evaluation_params=[
        "Employee email collected/inferred",
        "Laptop model selected",
        "Manager approval confirmed (if required)",
        "No redundant questions asked"
    ],
    evaluation_steps=[
        "Check if employee email is in conversation state",
        "Check if laptop model was selected (1, 2, or 3)",
        "Verify ticket created only after all info collected"
    ],
    threshold=1.0  # Must collect ALL info (100%)
)

# Test case
test_case = LLMTestCase(
    input="I need a new laptop",
    actual_output=agent_conversation,
    expected_output=None,
    retrieval_context=conversation_state
)

# Evaluate
score = info_gathering.measure(test_case)
# score = 1.0 (PASS) - All required info collected
# score = 0.75 (FAIL) - Missing laptop model selection
```

**3. Ticket Number Format Metric**

```python
import re
from deepeval.metrics import BaseMetric

class TicketFormatMetric(BaseMetric):
    """Verify ServiceNow ticket number format"""
    
    def __init__(self, threshold=1.0):
        self.threshold = threshold
        
    def measure(self, test_case: LLMTestCase):
        conversation = test_case.actual_output
        
        # Extract ticket numbers from conversation
        ticket_pattern = r'RITM\d{7}'
        tickets = re.findall(ticket_pattern, conversation)
        
        if len(tickets) == 0:
            # No ticket created
            return 0.0
        elif len(tickets) == 1:
            # Valid ticket format
            return 1.0
        else:
            # Multiple tickets (error)
            return 0.0
    
    def is_successful(self):
        return self.score >= self.threshold

# Usage
ticket_metric = TicketFormatMetric()
score = ticket_metric.measure(test_case)
# score = 1.0 (PASS) - "Ticket RITM0012345 created"
# score = 0.0 (FAIL) - "Ticket 12345 created" (wrong format)
```

**4. Tool Calling Accuracy Metric**

```python
from deepeval.metrics import BaseMetric

class ToolCallingMetric(BaseMetric):
    """Verify agent called correct MCP tools with correct arguments"""
    
    def __init__(self, expected_tools: list[dict]):
        self.expected_tools = expected_tools
        
    def measure(self, test_case: LLMTestCase):
        # Get actual tool calls from conversation state
        actual_tools = test_case.retrieval_context.get("tool_calls", [])
        
        score = 0
        for expected in self.expected_tools:
            # Find matching tool call
            match = next(
                (t for t in actual_tools if t["name"] == expected["name"]),
                None
            )
            
            if match:
                # Check arguments
                if match["args"] == expected["args"]:
                    score += 1
                elif self._args_close_enough(match["args"], expected["args"]):
                    score += 0.8
        
        return score / len(self.expected_tools)

# Usage
expected_tools = [
    {
        "name": "get_employee_by_email",
        "args": {"email": "john.doe@example.com"}
    },
    {
        "name": "create_ticket",
        "args": {
            "user_email": "john.doe@example.com",
            "ticket_type": "RITM",
            "description": "Laptop refresh: ThinkPad X1 Carbon Gen 11"
        }
    }
]

tool_metric = ToolCallingMetric(expected_tools)
score = tool_metric.measure(test_case)
# score = 1.0 (PASS) - All tools called correctly
# score = 0.5 (FAIL) - Only 1 of 2 tools correct
```

**5. Conversational Quality Metric**

```python
from deepeval.metrics import ConversationalRelevancyMetric

conversation_quality = ConversationalRelevancyMetric(
    model="gpt-4o",  # LLM-as-judge
    threshold=0.7
)

# Evaluates:
# - Are agent responses relevant to user questions?
# - Is conversation coherent across turns?
# - Does agent stay on topic?

score = conversation_quality.measure(test_case)
# Uses GPT-4o to judge conversation quality
```

---

### 2.3 Complete Evaluation Suite

**Red Hat's 15+ Metrics:**

| Metric | Type | Purpose | Threshold |
|--------|------|---------|-----------|
| **Policy Compliance** | Business Logic | Enforces 3-year policy | 0.8 |
| **Info Gathering Completeness** | Business Logic | All required fields | 1.0 |
| **Ticket Number Format** | Technical | Valid RITM format | 1.0 |
| **Tool Calling Accuracy** | Technical | Correct MCP calls | 0.9 |
| **Conversational Relevancy** | Quality | On-topic responses | 0.7 |
| **Answer Relevancy** | Quality | Answers user question | 0.7 |
| **Faithfulness** | Quality | Grounded in context | 0.8 |
| **Contextual Precision** | Quality | Accurate context usage | 0.7 |
| **Contextual Recall** | Quality | Uses all relevant context | 0.7 |
| **Hallucination** | Safety | No fabricated info | 0.9 |
| **Toxicity** | Safety | No harmful content | 1.0 |
| **Bias** | Safety | No discriminatory content | 0.9 |
| **Response Time** | Performance | <5s per turn | Pass/Fail |
| **Conversation Length** | Efficiency | <10 turns for ticket | Pass/Fail |
| **Success Rate** | Outcome | Ticket created | 1.0 |

**Passing Criteria:**
```python
# Test passes if ALL of:
- policy_compliance >= 0.8
- info_gathering == 1.0
- ticket_format == 1.0
- tool_calling >= 0.9
- conversational_relevancy >= 0.7
- hallucination >= 0.9
- toxicity == 1.0
- success_rate == 1.0
```

---

### 2.4 Test Dataset Structure

**Golden Test Conversations:**

```
tests/evaluation/
├── test_cases/
│   ├── laptop_refresh/
│   │   ├── eligible_simple.json
│   │   ├── eligible_complex.json
│   │   ├── not_eligible.json
│   │   ├── edge_case_exactly_3_years.json
│   │   ├── multi_request.json
│   │   └── error_handling.json
│   ├── password_reset/
│   │   └── ...
│   └── access_request/
│       └── ...
├── synthetic/
│   └── generated_conversations.json
└── production/
    └── exported_conversations.json
```

**Example Test Case:**

```json
{
  "test_id": "laptop_eligible_simple",
  "description": "User requests laptop, is eligible (4-year-old laptop)",
  "agent": "laptop_refresh_agent",
  "conversation": [
    {
      "role": "user",
      "content": "I need a new laptop",
      "channel": "slack"
    },
    {
      "role": "assistant",
      "content": "Hi! Let me check your eligibility...",
      "tool_calls": [
        {
          "name": "get_employee_by_email",
          "args": {"email": "john.doe@example.com"}
        }
      ]
    },
    {
      "role": "tool",
      "name": "get_employee_by_email",
      "content": "{\"name\": \"John Doe\", \"laptop_model\": \"ThinkPad X1 Gen 7\", \"laptop_purchase_date\": \"2022-01-15\"}"
    },
    {
      "role": "assistant",
      "content": "You're eligible! Your laptop is 4 years old. Available models: 1. ThinkPad, 2. MacBook, 3. Dell"
    },
    {
      "role": "user",
      "content": "1"
    },
    {
      "role": "assistant",
      "content": "Ticket RITM0012345 created!",
      "tool_calls": [
        {
          "name": "create_ticket",
          "args": {
            "user_email": "john.doe@example.com",
            "ticket_type": "RITM",
            "model": "ThinkPad X1 Carbon Gen 11"
          }
        }
      ]
    }
  ],
  "expected_outcome": {
    "success": true,
    "ticket_created": true,
    "ticket_format": "RITM\\d{7}",
    "policy_followed": true,
    "info_complete": true
  },
  "metrics": {
    "policy_compliance": {"threshold": 0.8},
    "info_gathering": {"threshold": 1.0},
    "ticket_format": {"threshold": 1.0},
    "tool_calling": {
      "threshold": 0.9,
      "expected_tools": [
        {"name": "get_employee_by_email", "args": {"email": "john.doe@example.com"}},
        {"name": "create_ticket", "args": {"user_email": "john.doe@example.com", "ticket_type": "RITM"}}
      ]
    }
  }
}
```

---

### 2.5 Running Evaluations

**Command Line:**

```bash
# Run all evaluation tests
pytest tests/evaluation/test_laptop_refresh.py --deepeval

# Run specific test
pytest tests/evaluation/test_laptop_refresh.py::test_eligible_simple --deepeval

# Generate synthetic tests
deepeval generate-test-cases \
  --agent laptop_refresh_agent \
  --num-cases 50 \
  --output tests/evaluation/synthetic/

# Re-evaluate production conversations
deepeval evaluate \
  --conversations production/jan_2026.json \
  --metrics policy_compliance,info_gathering,tool_calling
```

**Test Output:**

```
============================================ DEEPEVAL =============================================

Test: test_eligible_simple
Agent: laptop_refresh_agent
Metrics:
  ✅ Policy Compliance:             0.95 (threshold: 0.8)  PASS
  ✅ Info Gathering:                1.00 (threshold: 1.0)  PASS
  ✅ Ticket Format:                 1.00 (threshold: 1.0)  PASS
  ✅ Tool Calling Accuracy:         1.00 (threshold: 0.9)  PASS
  ✅ Conversational Relevancy:      0.85 (threshold: 0.7)  PASS
  ✅ Hallucination:                 1.00 (threshold: 0.9)  PASS
  ⏱️  Response Time:                3.2s (threshold: <5s)  PASS

Overall: PASS (7/7 metrics passed)

Test: test_not_eligible
Agent: laptop_refresh_agent
Metrics:
  ✅ Policy Compliance:             0.90 (threshold: 0.8)  PASS
  ❌ Info Gathering:                0.80 (threshold: 1.0)  FAIL - Missing justification field
  ✅ Ticket Format:                 N/A (no ticket created)
  ✅ Tool Calling Accuracy:         1.00 (threshold: 0.9)  PASS
  ✅ Conversational Relevancy:      0.75 (threshold: 0.7)  PASS

Overall: FAIL (1/5 metrics failed)

============================================ SUMMARY ===============================================
Total Tests: 15
Passed: 13
Failed: 2
Pass Rate: 86.7%

Failed Tests:
  - test_not_eligible: Info Gathering metric failed
  - test_error_handling: Response Time metric failed (8.2s > 5s threshold)
```

---

### 2.6 Synthetic Test Generation

**LLM-Generated Test Cases:**

```python
from deepeval.synthesizer import Synthesizer

synthesizer = Synthesizer(
    model="gpt-4o",
    agent_description="""
    You are an IT support agent that helps employees with laptop refresh requests.
    You check eligibility (3-year policy), collect laptop model preference,
    and create ServiceNow tickets.
    """
)

# Generate test cases
test_cases = synthesizer.generate_goldens(
    num_goldens=50,
    include_expected_output=True,
    context=[
        "Company policy: Laptops eligible for refresh after 3 years",
        "Available models: ThinkPad X1, MacBook Pro 14, Dell XPS 13",
        "Tickets created in ServiceNow with format RITM0012345"
    ]
)

# Generated test cases include:
# - Happy path (eligible user)
# - Edge cases (exactly 3 years, new employee)
# - Error scenarios (ServiceNow down, user not found)
# - Multi-request (laptop + password reset)
# - Ambiguous requests ("I need help")
```

**Example Generated Test:**

```json
{
  "user_input": "My laptop is really old and slow, can I get a new one?",
  "expected_output": "Agent checks laptop age via ServiceNow, finds it's 5 years old, approves request, offers model choices, creates ticket after user selects model",
  "expected_tools": ["get_employee_by_email", "create_ticket"],
  "edge_case": false,
  "difficulty": "easy"
}
```

---

## 3. NVIDIA Evaluation Gap

### 3.1 Current State

**What NVIDIA Has:**

```
✅ LangSmith Integration (Optional)
   - Traces agent execution
   - Shows conversation flow
   - Debugging/inspection only
   - No automated evaluation

✅ Manual Testing
   - Developers test via Gradio UI
   - Ad-hoc voice testing
   - No systematic coverage

✅ Logs
   - Agent outputs logged to stdout
   - Can review conversations manually
   - No metrics extracted
```

**What NVIDIA Lacks:**

```
❌ Automated Evaluation Framework
   - No DeepEval or equivalent
   - No systematic metrics

❌ Test Dataset
   - No golden conversations
   - No regression tests

❌ Business Metrics
   - No "intake completeness" metric
   - No "HIPAA compliance" checks
   - No "empathy score"

❌ CI/CD Integration
   - No automated testing in pipeline
   - No quality gates

❌ Production Validation
   - No ongoing evaluation of live conversations
   - No drift detection
```

---

### 3.2 Impact of Evaluation Gap

**Risks Without Evaluation:**

| Risk | Description | Impact |
|------|-------------|--------|
| **Silent Regressions** | Code change breaks agent, not detected until production | Users encounter failures |
| **Policy Violations** | Agent gives wrong medical advice, no one catches it | Liability, patient harm |
| **Degraded UX** | Agent becomes less helpful over time (prompt drift) | User satisfaction drops |
| **HIPAA Non-Compliance** | Agent leaks PHI, no automated detection | Legal/compliance risk |
| **Inconsistent Quality** | Agent quality varies by specialist, no metrics to compare | Uneven user experience |

**Example Regression Scenario:**

```
Developer Updates Patient Intake Agent
├─ Changes system prompt to "be more concise"
├─ No evaluation → Deploys to production
└─ Production Impact:
    ├─ Agent now skips allergy questions (too concise!)
    ├─ Incomplete patient records
    ├─ Discovered weeks later when nurse notices
    └─ Would have been caught by "Info Gathering Metric"
```

---

### 3.3 What NVIDIA Should Add

**Recommended Evaluation Metrics for Healthcare:**

```python
# 1. Clinical Completeness
clinical_completeness = GEval(
    name="Clinical Completeness",
    criteria="Agent collects all required clinical information",
    evaluation_params=[
        "Patient name collected",
        "Date of birth collected",
        "Current medications collected",
        "Allergies collected",
        "Chief complaint/symptoms collected"
    ],
    threshold=1.0  # Must collect ALL
)

# 2. HIPAA Compliance
hipaa_compliance = GEval(
    name="HIPAA Compliance",
    criteria="Agent handles PHI appropriately",
    evaluation_params=[
        "No PHI in logs (check for SSN, credit card)",
        "Patient data saved to FHIR (not local files)",
        "Secure communication (HTTPS/TLS)",
        "No unauthorized data sharing"
    ],
    threshold=1.0  # Zero tolerance
)

# 3. Empathy & Bedside Manner
empathy_metric = GEval(
    name="Empathy",
    criteria="Agent demonstrates appropriate empathy and bedside manner",
    evaluation_params=[
        "Acknowledges patient concerns",
        "Uses warm, professional language",
        "Shows patience with confused/elderly patients",
        "Offers reassurance appropriately"
    ],
    threshold=0.7
)

# 4. Medical Accuracy
medical_accuracy = GEval(
    name="Medical Accuracy",
    criteria="Agent provides accurate medical information",
    evaluation_params=[
        "Medication names spelled correctly",
        "Allergy information accurate",
        "No medical advice given (stays in lane)",
        "Refers to healthcare provider when appropriate"
    ],
    threshold=0.9
)

# 5. Safety & Escalation
safety_escalation = GEval(
    name="Safety Escalation",
    criteria="Agent recognizes and escalates emergency situations",
    evaluation_params=[
        "Detects emergency keywords (chest pain, difficulty breathing)",
        "Immediately instructs patient to call 911",
        "Does not attempt to diagnose or treat",
        "Alerts staff for in-person emergencies"
    ],
    threshold=1.0  # Critical
)
```

**Test Cases NVIDIA Should Create:**

```
tests/evaluation/
├── patient_intake/
│   ├── complete_intake_simple.json
│   ├── complete_intake_complex.json (multiple medications)
│   ├── elderly_patient.json (slow speech, confusion)
│   ├── non_english_speaker.json (accent, language barrier)
│   ├── emergency_chest_pain.json (should escalate)
│   ├── incomplete_info.json (patient doesn't know DOB)
│   └── privacy_concern.json (patient asks about privacy)
├── appointment_scheduling/
│   └── ...
└── medication_lookup/
    └── ...
```

---

## 4. Testing Strategies

### 4.1 Test Pyramid for Agents

```
                    ▲
                   ╱ ╲
                  ╱   ╲
                 ╱ E2E ╲          Few, Slow, Expensive
                ╱ Tests ╲         (Full agent conversations)
               ╱─────────╲
              ╱           ╲
             ╱ Integration ╲      More, Medium Speed
            ╱     Tests     ╲     (Agent + Tools)
           ╱───────────────╲
          ╱                 ╲
         ╱   Unit Tests      ╲    Many, Fast, Cheap
        ╱ (Tool functions,    ╲   (Pure Python)
       ╱   Prompt templates)   ╲
      ╱─────────────────────────╲
```

---

### 4.2 Unit Tests (Fast, Many)

**Red Hat Example:**

```python
# tests/unit/test_mcp_tools.py

import pytest
from mcp_servers.snow import get_employee_by_email

def test_get_employee_by_email_success():
    """Test ServiceNow employee lookup"""
    
    # Mock ServiceNow API
    with mock_servicenow():
        result = get_employee_by_email("john.doe@example.com")
    
    assert result["name"] == "John Doe"
    assert result["laptop_model"] == "ThinkPad X1 Gen 7"
    assert "laptop_purchase_date" in result

def test_get_employee_by_email_not_found():
    """Test employee not found scenario"""
    
    with mock_servicenow(status=404):
        with pytest.raises(EmployeeNotFoundError):
            get_employee_by_email("nonexistent@example.com")

def test_create_ticket_valid():
    """Test ticket creation with valid data"""
    
    with mock_servicenow():
        ticket = create_ticket(
            user_email="john.doe@example.com",
            ticket_type="RITM",
            description="Laptop refresh"
        )
    
    assert ticket["number"].startswith("RITM")
    assert len(ticket["number"]) == 12  # RITM + 7 digits

# Run: pytest tests/unit/ -v
# Fast: <1s per test, 100+ tests run in <10s
```

**NVIDIA Example:**

```python
# tests/unit/test_fhir_tools.py

import pytest
from tools.fhir import get_patient_medications

def test_get_patient_medications_success():
    """Test FHIR medication lookup"""
    
    with mock_fhir_server():
        meds = get_patient_medications("patient-123")
    
    assert len(meds) > 0
    assert "Lisinopril" in [m["name"] for m in meds]

def test_get_patient_medications_empty():
    """Test patient with no medications"""
    
    with mock_fhir_server(medications=[]):
        meds = get_patient_medications("patient-456")
    
    assert meds == []

# tests/unit/test_prompts.py

def test_patient_intake_prompt_formatting():
    """Test system prompt includes all required fields"""
    
    from prompts import PATIENT_INTAKE_SYSTEM_PROMPT
    
    assert "name" in PATIENT_INTAKE_SYSTEM_PROMPT
    assert "date of birth" in PATIENT_INTAKE_SYSTEM_PROMPT
    assert "allergies" in PATIENT_INTAKE_SYSTEM_PROMPT
    assert "medications" in PATIENT_INTAKE_SYSTEM_PROMPT
```

---

### 4.3 Integration Tests (Medium)

**Red Hat Example:**

```python
# tests/integration/test_laptop_refresh_agent.py

import pytest
from agent_service import laptop_refresh_agent

@pytest.mark.integration
def test_laptop_refresh_eligible_integration():
    """Test agent with real MCP servers (mocked ServiceNow)"""
    
    # Start MCP servers
    with mcp_test_harness():
        # Mock ServiceNow responses
        mock_employee_data({
            "email": "john.doe@example.com",
            "laptop_age": 4
        })
        
        # Run agent
        state = {
            "messages": [{"role": "user", "content": "I need a laptop"}],
            "user_email": "john.doe@example.com"
        }
        
        result = laptop_refresh_agent(state)
        
        # Assertions
        assert "eligible" in result["messages"][-1]["content"].lower()
        assert "ThinkPad" in result["messages"][-1]["content"]
        assert result.get("ticket_id") is not None

# Run: pytest tests/integration/ -m integration -v
# Medium speed: ~5s per test, 20 tests in ~2 min
```

**NVIDIA Example:**

```python
# tests/integration/test_patient_intake_agent.py

@pytest.mark.integration
def test_patient_intake_with_tools():
    """Test patient intake agent with FHIR/database tools"""
    
    with mock_fhir(), mock_database():
        # Simulate conversation
        state = {"messages": []}
        
        # Turn 1: User states intent
        state["messages"].append({"role": "user", "content": "I'm here for my appointment"})
        result = patient_intake_agent(state)
        
        # Turn 2: User provides name
        state["messages"].append({"role": "user", "content": "John Doe"})
        result = patient_intake_agent(state)
        
        # Turn 3: User provides DOB
        state["messages"].append({"role": "user", "content": "March 15, 1985"})
        result = patient_intake_agent(state)
        
        # Verify patient info tool was called
        tool_calls = [m for m in result["messages"] if hasattr(m, "tool_calls")]
        assert any("print_gathered_patient_info" in str(tc) for tc in tool_calls)
```

---

### 4.4 End-to-End Tests (Slow, Few)

**Red Hat Example:**

```python
# tests/e2e/test_full_workflow.py

@pytest.mark.e2e
@pytest.mark.slow
def test_laptop_refresh_full_workflow():
    """Test complete laptop refresh workflow end-to-end"""
    
    # Start all services (Request Manager, Agent, MCP servers, Kafka)
    with full_stack_deployment():
        
        # 1. Simulate Slack message
        slack_event = {
            "type": "message",
            "user": "U123",
            "text": "@IT-Agent I need a laptop",
            "channel": "C456"
        }
        
        response = requests.post(
            "http://integration-dispatcher:8080/slack/events",
            json=slack_event
        )
        
        # 2. Wait for agent processing (async via Kafka)
        time.sleep(5)
        
        # 3. Verify response sent to Slack
        slack_responses = get_slack_mock_responses()
        assert len(slack_responses) > 0
        assert "eligible" in slack_responses[0]["text"]
        
        # 4. User selects laptop
        slack_event_2 = {
            "type": "message",
            "user": "U123",
            "text": "1",
            "channel": "C456"
        }
        
        response = requests.post(
            "http://integration-dispatcher:8080/slack/events",
            json=slack_event_2
        )
        
        time.sleep(5)
        
        # 5. Verify ticket created in ServiceNow
        tickets = get_servicenow_mock_tickets()
        assert len(tickets) == 1
        assert tickets[0]["number"].startswith("RITM")

# Run: pytest tests/e2e/ -m e2e -v
# Slow: ~30s per test, 5 tests in ~3 min
```

**NVIDIA Example:**

```python
# tests/e2e/test_voice_conversation.py

@pytest.mark.e2e
@pytest.mark.slow
def test_voice_patient_intake_e2e():
    """Test complete voice conversation end-to-end"""
    
    # Start all services (FastAPI, RIVA ASR/TTS, Agent)
    with full_voice_stack():
        
        # 1. Establish WebRTC connection
        client = WebRTCTestClient("http://localhost:8081")
        client.connect()
        
        # 2. Send audio: "I'm here for my appointment"
        audio_1 = load_test_audio("here_for_appointment.wav")
        client.send_audio(audio_1)
        
        # 3. Wait for agent response
        response_1 = client.wait_for_audio(timeout=10)
        transcript_1 = transcribe_audio(response_1)
        assert "name" in transcript_1.lower()
        
        # 4. Send audio: "John Doe"
        audio_2 = load_test_audio("john_doe.wav")
        client.send_audio(audio_2)
        
        # ... continue conversation ...
        
        # 10. Verify patient_info.json created
        assert os.path.exists("/output/patient_info.json")
        data = json.load(open("/output/patient_info.json"))
        assert data["patient_name"] == "John Doe"

# Run: pytest tests/e2e/ -m e2e --slow -v
# Very slow: ~60s per test, requires full stack
```

---

### 4.5 Test Coverage Summary

| Test Type | NVIDIA | Red Hat | Analysis |
|-----------|--------|---------|----------|
| **Unit Tests** | Some (tools, prompts) | Comprehensive (MCP, utils) | Red Hat better coverage |
| **Integration Tests** | Limited | Good (agent + MCP) | Red Hat has more |
| **E2E Tests** | Manual only | Automated via DeepEval | Red Hat automated |
| **Evaluation Tests** | None | 15+ metrics | Red Hat only |
| **Load Tests** | None mentioned | Likely present | Unknown |
| **Security Tests** | None mentioned | Likely present | Unknown |

---

## 5. GitOps & CI/CD Pipelines

### 5.1 Red Hat CI/CD Pipeline

**GitHub Actions Workflow:**

```yaml
# .github/workflows/main.yml

name: IT Self-Service Agent CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  
  # ============================================
  # Stage 1: Linting & Static Analysis
  # ============================================
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install ruff mypy black
      
      - name: Run Ruff (linter)
        run: ruff check .
      
      - name: Run Black (formatter check)
        run: black --check .
      
      - name: Run MyPy (type checking)
        run: mypy agent_service/ mcp_servers/
  
  # ============================================
  # Stage 2: Unit Tests
  # ============================================
  unit-tests:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov
      
      - name: Run Unit Tests
        run: |
          pytest tests/unit/ \
            --cov=agent_service \
            --cov=mcp_servers \
            --cov-report=xml \
            --cov-report=term \
            --cov-fail-under=80
      
      - name: Upload Coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.xml
  
  # ============================================
  # Stage 3: Integration Tests
  # ============================================
  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      kafka:
        image: confluentinc/cp-kafka:latest
        env:
          KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Start MCP Test Servers
        run: |
          docker compose -f docker-compose.test.yml up -d mcp-servers
      
      - name: Run Integration Tests
        run: |
          pytest tests/integration/ \
            -m integration \
            --timeout=300
      
      - name: Cleanup
        run: docker compose -f docker-compose.test.yml down
  
  # ============================================
  # Stage 4: Agent Evaluation (DeepEval)
  # ============================================
  evaluation:
    runs-on: ubuntu-latest
    needs: integration-tests
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install DeepEval
        run: pip install deepeval pytest
      
      - name: Start Agent Service
        run: |
          docker compose up -d agent-service llama-stack
          sleep 30  # Wait for services to be ready
      
      - name: Run DeepEval Tests
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          pytest tests/evaluation/ \
            --deepeval \
            --timeout=600 \
            --junitxml=evaluation-results.xml
      
      - name: Upload Evaluation Results
        uses: actions/upload-artifact@v3
        with:
          name: evaluation-results
          path: evaluation-results.xml
      
      - name: Check Evaluation Pass Rate
        run: |
          # Fail if pass rate < 90%
          python scripts/check_eval_pass_rate.py \
            evaluation-results.xml \
            --threshold 0.90
  
  # ============================================
  # Stage 5: Build & Push Container Images
  # ============================================
  build:
    runs-on: ubuntu-latest
    needs: evaluation
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Login to Container Registry
        uses: docker/login-action@v2
        with:
          registry: quay.io
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}
      
      - name: Build and Push Agent Service
        uses: docker/build-push-action@v4
        with:
          context: ./agent-service
          push: true
          tags: |
            quay.io/redhat/it-agent:${{ github.sha }}
            quay.io/redhat/it-agent:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Build and Push MCP Servers
        uses: docker/build-push-action@v4
        with:
          context: ./mcp-servers
          push: true
          tags: quay.io/redhat/it-agent-mcp:${{ github.sha }}
  
  # ============================================
  # Stage 6: Deploy to Staging
  # ============================================
  deploy-staging:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment: staging
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up kubectl
        uses: azure/setup-kubectl@v3
      
      - name: Set up Helm
        uses: azure/setup-helm@v3
      
      - name: Deploy to Staging OpenShift
        env:
          KUBECONFIG: ${{ secrets.STAGING_KUBECONFIG }}
        run: |
          helm upgrade --install it-agent ./helm/it-agent \
            --namespace it-agent-staging \
            --create-namespace \
            --set image.tag=${{ github.sha }} \
            --set environment=staging \
            --wait \
            --timeout 10m
      
      - name: Run Smoke Tests
        run: |
          kubectl run smoke-test \
            --namespace it-agent-staging \
            --image=curlimages/curl \
            --restart=Never \
            --rm -i \
            --command -- \
            curl http://agent-service:8080/health
  
  # ============================================
  # Stage 7: Deploy to Production (Manual Approval)
  # ============================================
  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    environment: production  # Requires manual approval in GitHub
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Production OpenShift
        env:
          KUBECONFIG: ${{ secrets.PRODUCTION_KUBECONFIG }}
        run: |
          helm upgrade --install it-agent ./helm/it-agent \
            --namespace it-agent-prod \
            --create-namespace \
            --set image.tag=${{ github.sha }} \
            --set environment=production \
            --set replicas=3 \
            --wait \
            --timeout 15m
      
      - name: Notify Slack
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "IT Agent deployed to production: ${{ github.sha }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

### 5.2 NVIDIA CI/CD Pipeline (Hypothetical - Should Add)

**GitHub Actions Workflow:**

```yaml
# .github/workflows/main.yml (NVIDIA should add this)

name: Ambient Patient Agent CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  
  # ============================================
  # Stage 1: Linting
  # ============================================
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - run: pip install ruff black
      - run: ruff check .
      - run: black --check .
  
  # ============================================
  # Stage 2: Unit Tests
  # ============================================
  unit-tests:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v3
      - run: pip install -r requirements.txt pytest
      - run: pytest tests/unit/ --cov=tools --cov=agents
  
  # ============================================
  # Stage 3: Agent Evaluation (SHOULD ADD)
  # ============================================
  evaluation:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v3
      
      - name: Install DeepEval
        run: pip install deepeval pytest
      
      - name: Start Services (Mock RIVA, LLM)
        run: |
          docker compose -f docker-compose.test.yml up -d
          sleep 30
      
      - name: Run Healthcare Evaluation
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          pytest tests/evaluation/ \
            --deepeval \
            --junitxml=eval-results.xml
      
      - name: Check Pass Rate
        run: |
          python scripts/check_eval_pass_rate.py \
            eval-results.xml \
            --threshold 0.85
  
  # ============================================
  # Stage 4: Build Container Images
  # ============================================
  build:
    runs-on: ubuntu-latest
    needs: evaluation
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      
      - name: Build FastAPI Server
        run: |
          docker build -t nvcr.io/nvidia/ambient-patient:${{ github.sha }} .
      
      - name: Push to NGC
        run: |
          docker login nvcr.io -u '$oauthtoken' -p ${{ secrets.NGC_API_KEY }}
          docker push nvcr.io/nvidia/ambient-patient:${{ github.sha }}
  
  # ============================================
  # Stage 5: Deploy to Staging
  # ============================================
  deploy-staging:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Deploy to Kubernetes
        run: |
          helm upgrade --install ambient-patient ./helm \
            --namespace ambient-patient-staging \
            --set image.tag=${{ github.sha }} \
            --wait
```

---

### 5.3 GitOps Comparison

| Aspect | NVIDIA (Current) | Red Hat | Analysis |
|--------|------------------|---------|----------|
| **CI/CD Pipeline** | ❌ Not present | ✅ Comprehensive (7 stages) | Red Hat production-ready |
| **Automated Testing** | ❌ Manual only | ✅ Unit + Integration + E2E | Red Hat has quality gates |
| **Evaluation in CI** | ❌ None | ✅ DeepEval in pipeline | Red Hat catches regressions |
| **Container Builds** | Manual | ✅ Automated (GitHub Actions) | Red Hat automated |
| **Deployment** | Manual (docker compose) | ✅ Helm to OpenShift | Red Hat GitOps |
| **Environments** | Single (dev) | ✅ Staging + Production | Red Hat has proper promotion |
| **Rollback** | Manual | ✅ Helm rollback | Red Hat safer |
| **Secrets Management** | .env files | ✅ Kubernetes Secrets | Red Hat more secure |

---

## 6. Deployment Strategies

### 6.1 Red Hat Deployment Strategy

**Environments:**

```
Development → Staging → Production
    ↓           ↓           ↓
  Local     OpenShift   OpenShift
  Docker    (test)      (prod)
```

**Helm Chart Structure:**

```
helm/it-agent/
├── Chart.yaml
├── values.yaml              # Default values
├── values-staging.yaml      # Staging overrides
├── values-production.yaml   # Production overrides
├── templates/
│   ├── agent-service/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── hpa.yaml         # Horizontal Pod Autoscaler
│   │   └── configmap.yaml
│   ├── mcp-servers/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── request-manager/
│   │   └── ...
│   ├── integration-dispatcher/
│   │   └── ...
│   ├── postgres/
│   │   ├── statefulset.yaml
│   │   └── pvc.yaml
│   └── kafka/
│       └── ...
└── README.md
```

**values-production.yaml:**

```yaml
# Production overrides
environment: production

agent-service:
  replicas: 3  # High availability
  resources:
    requests:
      memory: "4Gi"
      cpu: "2"
    limits:
      memory: "8Gi"
      cpu: "4"
  
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 10
    targetCPUUtilizationPercentage: 70

llama-stack:
  replicas: 2
  gpu:
    enabled: true
    count: 4  # 4x A100 per pod
    type: nvidia.com/gpu

postgres:
  replicas: 3  # HA cluster
  persistence:
    size: 100Gi
    storageClass: fast-ssd

kafka:
  replicas: 3
  persistence:
    size: 500Gi

monitoring:
  enabled: true
  opentelemetry: true
  jaeger: true
  langfuse: true

secrets:
  # Secrets managed via Kubernetes Secrets
  servicenow:
    secretName: servicenow-credentials
  slack:
    secretName: slack-credentials
  smtp:
    secretName: smtp-credentials
```

**Deployment Command:**

```bash
# Deploy to production
helm upgrade --install it-agent ./helm/it-agent \
  --namespace it-agent-prod \
  --values helm/it-agent/values-production.yaml \
  --set image.tag=abc123def \
  --wait \
  --timeout 15m

# Rollback if needed
helm rollback it-agent -n it-agent-prod

# Check status
helm status it-agent -n it-agent-prod
kubectl get pods -n it-agent-prod
```

---

### 6.2 NVIDIA Deployment Strategy (Current)

**Deployment Method:**

```bash
# Current: Docker Compose (manual)
docker compose up -d

# Services:
# - agent-instruct-llm (Llama-3.3-70B NIM)
# - nemoguard-content-safety (NemoGuard-8B NIM)
# - nemoguard-topic-control (NemoGuard-8B NIM)
# - riva-asr (Parakeet ASR)
# - riva-tts (Magpie TTS)
# - ace-controller (FastAPI + Pipecat)
# - gradio-ui (Optional dev UI)
```

**Limitations:**

- ❌ No Kubernetes/Helm (not production-ready)
- ❌ Single instance (no HA)
- ❌ No autoscaling
- ❌ Manual updates
- ❌ No rollback mechanism
- ❌ .env file secrets (not secure for prod)

**What NVIDIA Should Add:**

```
Helm Chart:
  - Deployment for FastAPI server
  - StatefulSet for RIVA NIMs (GPU pods)
  - Service mesh (Istio) for traffic management
  - HPA for FastAPI (scale based on requests)
  - Persistent volumes for patient data
  - Secrets management (Kubernetes Secrets)
  - Ingress for HTTPS
  - Certificate management (cert-manager)
```

---

### 6.3 Blue-Green Deployment (Red Hat)

**Strategy:**

```
Production Namespace:
├── it-agent-blue (current production)
│   ├── 3 replicas
│   └── Serving 100% traffic
└── it-agent-green (new version)
    ├── 3 replicas
    └── Serving 0% traffic (initially)

Deployment Steps:
1. Deploy green version (new image tag)
2. Run smoke tests on green
3. Shift 10% traffic to green (canary)
4. Monitor metrics for 15 minutes
5. If successful, shift 50% → 100%
6. If failure, rollback to blue (instant)
7. Once stable, delete blue (cleanup)
```

**Traffic Shifting (Istio VirtualService):**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: it-agent
spec:
  hosts:
    - it-agent.example.com
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: it-agent-green
          weight: 100
    - route:
        - destination:
            host: it-agent-blue
          weight: 90  # 90% to blue (current)
        - destination:
            host: it-agent-green
          weight: 10  # 10% to green (canary)
```

---

## 7. Quality Gates & Release Criteria

### 7.1 Red Hat Quality Gates

**Before Merging to Main:**

```
✅ All linting passes (ruff, black, mypy)
✅ Unit test coverage >= 80%
✅ All integration tests pass
✅ DeepEval pass rate >= 90%
✅ No critical security vulnerabilities (Snyk scan)
✅ Code review approved by 2+ reviewers
```

**Before Deploying to Staging:**

```
✅ All CI checks pass on main branch
✅ Docker images built successfully
✅ Helm chart syntax valid (helm lint)
✅ No breaking changes in API
```

**Before Promoting to Production:**

```
✅ Staging deployment successful
✅ Smoke tests pass in staging
✅ Performance tests acceptable (<5s p95 latency)
✅ Load tests pass (1000 req/min sustained)
✅ Security scan clean (no critical/high vulns)
✅ Manual approval from team lead
✅ Deployment window approved (change management)
```

---

### 7.2 DeepEval as Quality Gate

**CI Configuration:**

```yaml
# In GitHub Actions
- name: Run DeepEval Tests
  run: pytest tests/evaluation/ --deepeval

- name: Check Evaluation Pass Rate
  run: |
    python scripts/check_eval_pass_rate.py \
      evaluation-results.xml \
      --threshold 0.90 \
      --fail-on-regression
```

**check_eval_pass_rate.py:**

```python
import sys
import xml.etree.ElementTree as ET

def check_pass_rate(results_file, threshold, fail_on_regression=False):
    """Check DeepEval pass rate and optionally detect regressions"""
    
    tree = ET.parse(results_file)
    root = tree.getroot()
    
    total = int(root.attrib['tests'])
    failures = int(root.attrib['failures'])
    errors = int(root.attrib['errors'])
    
    passed = total - failures - errors
    pass_rate = passed / total
    
    print(f"DeepEval Results:")
    print(f"  Total: {total}")
    print(f"  Passed: {passed}")
    print(f"  Failed: {failures}")
    print(f"  Errors: {errors}")
    print(f"  Pass Rate: {pass_rate:.2%}")
    
    if pass_rate < threshold:
        print(f"❌ FAIL: Pass rate {pass_rate:.2%} < threshold {threshold:.2%}")
        sys.exit(1)
    
    if fail_on_regression:
        # Compare with baseline (previous run)
        baseline = load_baseline_results()
        if pass_rate < baseline["pass_rate"] - 0.05:  # 5% regression
            print(f"❌ FAIL: Regression detected ({baseline['pass_rate']:.2%} → {pass_rate:.2%})")
            sys.exit(1)
    
    print(f"✅ PASS: Pass rate {pass_rate:.2%} >= threshold {threshold:.2%}")
    return pass_rate

if __name__ == "__main__":
    import argparse
    parser = argparse.ArgumentParser()
    parser.add_argument("results_file")
    parser.add_argument("--threshold", type=float, default=0.90)
    parser.add_argument("--fail-on-regression", action="store_true")
    args = parser.parse_args()
    
    check_pass_rate(args.results_file, args.threshold, args.fail_on_regression)
```

**Output:**

```
DeepEval Results:
  Total: 45
  Passed: 42
  Failed: 3
  Errors: 0
  Pass Rate: 93.33%

✅ PASS: Pass rate 93.33% >= threshold 90.00%
```

**If Fail:**

```
DeepEval Results:
  Total: 45
  Passed: 38
  Failed: 7
  Errors: 0
  Pass Rate: 84.44%

❌ FAIL: Pass rate 84.44% < threshold 90.00%

Failed Tests:
  - test_not_eligible: Info Gathering metric failed
  - test_edge_case_3_years: Policy Compliance metric failed
  - test_error_handling: Response Time exceeded 5s
  - ...

Pipeline BLOCKED. Fix failing tests before deploying.
```

---

## 8. Monitoring & Production Validation

### 8.1 Red Hat Production Monitoring

**OpenTelemetry Metrics:**

```python
from opentelemetry import metrics

meter = metrics.get_meter(__name__)

# Agent performance metrics
response_time = meter.create_histogram(
    "agent.response_time",
    unit="ms",
    description="Agent response time per turn"
)

tool_call_duration = meter.create_histogram(
    "mcp.tool_call_duration",
    unit="ms",
    description="MCP tool call duration"
)

conversation_length = meter.create_histogram(
    "agent.conversation_length",
    unit="turns",
    description="Number of turns per conversation"
)

success_rate = meter.create_counter(
    "agent.success_rate",
    description="Successful ticket creations"
)

# Record metrics
response_time.record(3200, {"agent": "laptop_refresh"})
tool_call_duration.record(250, {"tool": "get_employee_by_email"})
conversation_length.record(5, {"agent": "laptop_refresh", "outcome": "success"})
success_rate.add(1, {"agent": "laptop_refresh"})
```

**Prometheus Queries:**

```promql
# Average response time by agent
rate(agent_response_time_sum[5m]) / rate(agent_response_time_count[5m])

# Success rate
rate(agent_success_rate[5m])

# Tool call errors
rate(mcp_tool_call_errors[5m])

# p95 response time
histogram_quantile(0.95, rate(agent_response_time_bucket[5m]))
```

**Grafana Dashboard:**

```
┌─────────────────────────────────────────────────────────────┐
│  IT Agent Production Dashboard                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Response Time (p50/p95/p99)                                │
│  ▁▃▅▇█▇▅▃▁▃▅▇█▇▅▃▁ (graph showing 2.1s / 4.3s / 7.2s)     │
│                                                              │
│  Success Rate:  97.3% ✅                                     │
│  Conversations: 1,243 today                                 │
│  Tickets Created: 1,208                                     │
│                                                              │
│  Top Failure Reasons:                                       │
│  1. ServiceNow API timeout (12 failures)                    │
│  2. User not found (8 failures)                             │
│  3. LLM timeout (4 failures)                                │
│                                                              │
│  MCP Tool Performance:                                      │
│  - get_employee: 187ms avg                                  │
│  - create_ticket: 342ms avg                                 │
│  - search_knowledge: 523ms avg                              │
└─────────────────────────────────────────────────────────────┘
```

---

### 8.2 Continuous Evaluation in Production

**Langfuse Integration:**

```python
from langfuse import Langfuse

langfuse = Langfuse(
    public_key=os.getenv("LANGFUSE_PUBLIC_KEY"),
    secret_key=os.getenv("LANGFUSE_SECRET_KEY")
)

# Trace conversation
trace = langfuse.trace(
    name="laptop_refresh_conversation",
    user_id="john.doe@example.com",
    session_id=session_id,
    metadata={
        "channel": "slack",
        "agent": "laptop_refresh"
    }
)

# Trace each turn
for turn in conversation:
    trace.span(
        name="agent_turn",
        input=turn["input"],
        output=turn["output"],
        metadata={
            "tool_calls": turn.get("tool_calls", [])
        }
    )

# Score conversation after completion
trace.score(
    name="success",
    value=1.0 if ticket_created else 0.0
)

trace.score(
    name="user_satisfaction",
    value=user_feedback_score  # If user provides feedback
)
```

**Langfuse Dashboard:**

```
┌─────────────────────────────────────────────────────────────┐
│  Langfuse - Production Conversations                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Today's Conversations:  1,243                              │
│  Avg Success Score:      0.973                              │
│  Avg Conversation Length: 5.2 turns                         │
│                                                              │
│  Recent Conversations:                                      │
│  ┌────────────────────────────────────────────────────┐    │
│  │ session-abc123 | john.doe | 5 turns | ✅ Success  │    │
│  │ [View Trace] [Export] [Re-evaluate]                │    │
│  └────────────────────────────────────────────────────┘    │
│  ┌────────────────────────────────────────────────────┐    │
│  │ session-def456 | jane.smith | 3 turns | ❌ Failed  │    │
│  │ Reason: ServiceNow timeout                         │    │
│  │ [View Trace] [Export] [Debug]                      │    │
│  └────────────────────────────────────────────────────┘    │
│                                                              │
│  Export for Re-evaluation:                                  │
│  [Export Last 100] [Export Failed Only]                     │
└─────────────────────────────────────────────────────────────┘
```

**Re-evaluation of Production Data:**

```bash
# Export last week's conversations
langfuse export --days 7 --output prod_conversations_week12.json

# Re-evaluate with latest metrics
deepeval evaluate \
  --conversations prod_conversations_week12.json \
  --metrics policy_compliance,info_gathering,tool_calling \
  --output prod_eval_week12.html

# Compare with previous week
deepeval compare \
  --baseline prod_eval_week11.html \
  --current prod_eval_week12.html
```

**Detect Drift:**

```
Week 11 vs Week 12 Comparison:

Policy Compliance:     0.94 → 0.91 (⚠️  -3% regression)
Info Gathering:        0.98 → 0.97 (✅ stable)
Tool Calling:          0.96 → 0.94 (⚠️  -2% regression)
Conversational Quality: 0.82 → 0.84 (✅ +2% improvement)

⚠️  WARNING: Policy Compliance dropped 3%
   - Investigate recent prompt changes
   - Review failed cases manually
```

---

## 9. Continuous Improvement Loop

### 9.1 Red Hat Continuous Improvement

```
┌─────────────────────────────────────────────────────────────┐
│           Continuous Improvement Cycle                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Production Conversations                                │
│     ├─ Langfuse captures all traces                        │
│     └─ OpenTelemetry records metrics                       │
│                         ↓                                    │
│  2. Identify Issues                                         │
│     ├─ Low success rate conversations                      │
│     ├─ Failed DeepEval metrics                             │
│     └─ User feedback (negative ratings)                    │
│                         ↓                                    │
│  3. Root Cause Analysis                                     │
│     ├─ Review conversation traces                          │
│     ├─ Check tool call accuracy                            │
│     └─ Analyze prompt/model behavior                       │
│                         ↓                                    │
│  4. Create Test Cases                                       │
│     ├─ Export failing conversations                        │
│     ├─ Add to test suite as regression tests               │
│     └─ Document expected behavior                          │
│                         ↓                                    │
│  5. Implement Fix                                           │
│     ├─ Update prompts                                      │
│     ├─ Fix tool logic                                      │
│     └─ Adjust agent routing                                │
│                         ↓                                    │
│  6. Validate Fix                                            │
│     ├─ Run DeepEval locally                                │
│     ├─ Verify regression tests pass                        │
│     └─ Test on production data export                      │
│                         ↓                                    │
│  7. Deploy via CI/CD                                        │
│     ├─ All quality gates pass                              │
│     ├─ Staging deployment                                  │
│     └─ Production deployment (blue-green)                  │
│                         ↓                                    │
│  8. Monitor Impact                                          │
│     ├─ Track metrics (success rate, etc.)                  │
│     ├─ Re-evaluate with new data                           │
│     └─ Verify issue resolved                               │
│                         ↓                                    │
│  [Loop back to step 1]                                      │
└─────────────────────────────────────────────────────────────┘
```

**Example Improvement Cycle:**

```
Week 1: Issue Detected
├─ Policy Compliance metric drops to 85%
├─ Manual review: Agent not enforcing 3-year rule correctly
└─ Root cause: Prompt ambiguity around "exactly 3 years"

Week 1: Fix Implemented
├─ Update system prompt: "Eligible if age >= 3 years (inclusive)"
├─ Add test case: test_edge_case_exactly_3_years.json
└─ DeepEval shows 95% policy compliance locally

Week 2: Deployed to Production
├─ All CI checks pass
├─ Staging tests successful
└─ Production deployment via blue-green

Week 3: Impact Validated
├─ Policy Compliance back to 94%
├─ No new regressions
└─ Issue resolved ✅
```

---

### 9.2 NVIDIA Should Implement

**Recommended Improvement Loop:**

```
1. Capture Production Data
   - Add Langfuse tracing (currently missing)
   - Export all voice conversations with transcripts
   - Tag conversations (success/failure)

2. Create Evaluation Framework
   - Build DeepEval test suite (15+ metrics for healthcare)
   - Add clinical completeness, HIPAA compliance, empathy metrics
   - Automate evaluation in CI/CD

3. Establish Quality Gates
   - Require 85%+ evaluation pass rate before deploy
   - Block merges that cause regressions
   - Manual review for HIPAA-critical changes

4. Monitor Production
   - Track intake completeness rates
   - Monitor ASR accuracy (WER)
   - Measure user satisfaction

5. Iterate on Failures
   - Export failed conversations
   - Add as regression tests
   - Update prompts/flows

6. Fine-tune Models (Long-term)
   - Use production data to fine-tune Llama-3.3
   - Improve medical terminology understanding
   - Increase empathy scores
```

---

## 10. Recommendations

### 10.1 For NVIDIA (Critical Gaps)

**Priority 1: Add Evaluation Framework**

```bash
# Immediate action
1. Install DeepEval: pip install deepeval
2. Create tests/evaluation/ directory
3. Define 5 core metrics:
   - Clinical completeness
   - HIPAA compliance
   - Empathy
   - Medical accuracy
   - Safety escalation
4. Create 10 golden test conversations
5. Integrate into development workflow
```

**Priority 2: Add CI/CD Pipeline**

```yaml
# Add .github/workflows/main.yml
- Linting (ruff, black)
- Unit tests (pytest)
- Evaluation tests (deepeval)
- Build containers
- Deploy to staging
```

**Priority 3: Production Observability**

```python
# Add Langfuse tracing
from langfuse import Langfuse
# Trace all conversations
# Export for re-evaluation
```

**Priority 4: Quality Gates**

```
- Require 85% eval pass rate
- Block regressions
- Manual review for HIPAA changes
```

**Estimated Effort:**
- Evaluation framework: 2 weeks
- CI/CD pipeline: 1 week
- Langfuse integration: 1 week
- Quality gates: 1 week
- **Total: 5 weeks** to reach Red Hat's maturity

---

### 10.2 For Red Hat (Enhancements)

**Optimization 1: Faster Evaluation**

```python
# Current: Sequential evaluation (slow)
# Proposed: Parallel evaluation

from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=5) as executor:
    futures = [executor.submit(evaluate_test_case, tc) for tc in test_cases]
    results = [f.result() for f in futures]

# 5x speedup on evaluation time
```

**Optimization 2: Evaluation Caching**

```python
# Cache LLM-as-judge results (GPT-4o calls are expensive)
from functools import lru_cache

@lru_cache(maxsize=1000)
def evaluate_conversational_quality(conversation_hash):
    return gpt4o_judge(conversation)

# Avoid re-evaluating same conversations
```

**Optimization 3: A/B Testing Framework**

```python
# Test prompt variants in production
from abtest import ABTest

ab_test = ABTest(
    name="concise_prompts",
    variants={
        "control": original_prompt,
        "treatment": concise_prompt
    },
    traffic_split=0.5
)

# Measure which performs better on DeepEval metrics
```

---

### 10.3 Universal Best Practices

**1. Evaluation is Not Optional**

```
Without evaluation:
- Silent regressions
- Degrading quality over time
- No confidence in deployments

With evaluation:
- Catch issues before production
- Quantify improvements
- Data-driven decisions
```

**2. Test Production Data**

```
Golden datasets are not enough.
Re-evaluate production conversations weekly.
Detect drift early.
```

**3. Quality Gates Save Time**

```
Blocked bad deployment: 30 min
Debugging production issue: 4 hours
Rollback + hotfix: 2 hours
Customer trust lost: Priceless

Quality gates >> Incident response
```

**4. Automate Everything**

```
Manual testing does not scale.
CI/CD + automated evaluation = confidence at scale.
```

---

## 11. Key Takeaways

### 11.1 Evaluation Maturity

| Aspect | NVIDIA | Red Hat | Gap |
|--------|--------|---------|-----|
| **Evaluation Framework** | ❌ None | ✅ DeepEval (15+ metrics) | Critical |
| **Test Coverage** | Manual only | Automated (unit/integration/e2e) | Major |
| **CI/CD Pipeline** | ❌ None | ✅ 7-stage pipeline | Critical |
| **Quality Gates** | ❌ None | ✅ Pass rate thresholds | Major |
| **Production Monitoring** | Basic logs | ✅ OpenTelemetry + Langfuse | Major |
| **Continuous Improvement** | Ad-hoc | ✅ Systematic loop | Major |

**Overall:** Red Hat is **2+ years ahead** in evaluation/CI/CD maturity.

---

### 11.2 Production Readiness

**NVIDIA Current State:**
- ✅ Great technology (voice, NeMo Guardrails)
- ✅ Impressive demo
- ⚠️  Not production-ready (no evaluation, no CI/CD, no monitoring)
- 🎯 Focus: Prototype/demo quality

**Red Hat Current State:**
- ✅ Production-grade evaluation
- ✅ Comprehensive CI/CD
- ✅ Enterprise deployment (Helm, OpenShift)
- ✅ Continuous improvement loop
- 🎯 Focus: Enterprise production quality

---

### 11.3 Why This Matters

**For NVIDIA:**
```
Healthcare is high-stakes.
Agent gives wrong medical advice → Patient harm.
HIPAA violation → Legal liability.
Silent regression → Reputation damage.

Evaluation framework is not a nice-to-have.
It's a MUST-HAVE for healthcare.
```

**For Red Hat:**
```
IT automation at scale.
1000s of tickets/month.
Agent errors → Employee frustration.
Policy violations → Compliance issues.

Evaluation framework ensures quality at scale.
```

---

**Document Version:** 1.0  
**Created:** April 1, 2026  
**Companion to:** COMPARISON_ANALYSIS.md, STATE_MANAGEMENT_DEEP_DIVE.md, PROTOCOL_DEEP_DIVE.md, MODELS_DEEP_DIVE.md, UI_UX_DEEP_DIVE.md
