# Google Agent Development Kit (ADK) Complete Guide

## Introduction to ADK

Google's Agent Development Kit (ADK) provides a powerful framework for building AI agents that combines simplicity with sophisticated capabilities. At its core, ADK separates agent logic (what you write) from runtime infrastructure (what ADK handles), allowing developers to focus on business logic while the framework manages sessions, events, state persistence, and orchestration.

### Core Philosophy: Agent = Logic + Runtime
- **Agent Logic**: Your code defining the agent's purpose, instructions, and capabilities
- **Runtime**: ADK's infrastructure that handles sessions, events, and state persistence
- **You write the "what", ADK handles the "how"**

## Getting Started

### Minimal Agent Example
Start with just 7 lines of code:

```python
from google.adk.agents import Agent

root_agent = Agent(
    name="CustomerSupportAgent",
    model="gemini-2.0-flash",
    instruction="You help customers with their SmartHome products."
)
```

### Running Your Agent
ADK provides multiple interfaces for running agents:

```bash
adk run customer_support_agent    # Simple text interface
adk web                           # Rich web UI with debugging
adk web --reload_agents           # Auto-reload for development
adk api_server                    # REST API endpoint
```

**Pro Tip**: Use `adk web` during development for visual debugging - you can see tool calls, events, state changes, and the agent tree structure in real-time.

## Building Blocks

### 1. Tools - Extending Agent Capabilities

Tools are functions that agents can call for deterministic operations with clear inputs/outputs.

#### Creating Custom Tools

```python
def look_up_order(order_id: str) -> dict:
    """Retrieves order information from database.
    
    Args:
        order_id: The order ID to look up
    
    Returns:
        Dictionary containing order details
    """
    # Your logic here
    return {"status": "success", "order_data": order_data}

root_agent = Agent(
    name="CustomerSupportAgent",
    model="gemini-2.0-flash",
    instruction="...",
    tools=[look_up_order]  # ADK handles execution
)
```

#### Tool Best Practices
- **No default parameter values** - LLMs don't interpret them correctly
- **Return dictionaries** with a "status" key for consistency
- **Use JSON-serializable types only** (string, integer, list, dictionary)
- **Write comprehensive docstrings** - the LLM relies on these to understand usage
- **Keep tools focused** - single-purpose functions work best

#### Pre-Built Google Tools
- **Google Search** (`google_search`): Web searches
- **Code Execution** (`built_in_code_execution`): Execute code for calculations
- **Vertex AI Search** (`VertexAiSearchTool`): Search through AI Applications data stores

#### Third-Party Tool Integration

**LangChain Integration:**
```python
from google.adk.extensions import LangchainTool
from langchain_community.tools import WikipediaQueryRun

tools = [
    LangchainTool(
        tool=WikipediaQueryRun(api_wrapper=WikipediaAPIWrapper())
    )
]
```

**CrewAI Integration:**
```python
from google.adk.extensions import CrewaiTool
from crewai_tools import ScrapeWebsiteTool

tools = [
    CrewaiTool(
        name="scrape_news",
        description="Scrapes the latest news content",
        tool=ScrapeWebsiteTool("https://apnews.com/")
    )
]
```

### 2. Subagents - Delegating Complex Workflows

Subagents handle tasks requiring reasoning, context, or multi-step interactions.

```python
returns_agent = Agent(
    name="ReturnsSpecialist",
    instruction="Guide return processes with empathy...",
    tools=[validate_return, create_label]
)

root_agent = Agent(
    name="CustomerSupportAgent",
    instruction="Delegate return requests to ReturnsSpecialist",
    sub_agents=[returns_agent]  # Delegate to specialist
)
```

### Decision Matrix: Tools vs Subagents

| Factor | Use Tools | Use Subagents |
|--------|-----------|---------------|
| **Operation** | Single-purpose, deterministic | Multi-step, context-dependent |
| **Reasoning** | None needed | Understanding required |
| **Speed** | Fast, no token cost | Variable latency |
| **Interface** | Fixed inputs/outputs | Conversational |
| **Best For** | Database lookups, calculations, API calls | Complex workflows, specialized expertise, empathy/judgment |

## State Management

ADK provides three state scopes with clear purposes and lifecycles:

```python
async def manage_cart(product_id: str, tool_context: ToolContext):
    # Temporary: Session-only data
    tool_context.state["temp:verification_code"] = "123456"
    
    # User: Persists across sessions
    tool_context.state["user:cart"] = {"items": [...]}
    tool_context.state["user:preferences"] = {"shipping": "express"}
    
    # Application: System-wide settings (read-only in tools)
    threshold = tool_context.state.get("app:free_shipping_threshold", 50)
```

| Scope | Prefix | Lifetime | Use Cases |
|-------|--------|----------|-----------|
| **Temporary** | `temp:` | Current session | Form data, session metrics |
| **User** | `user:` | Forever (per user) | Preferences, history, cart |
| **Application** | `app:` | System-wide | Business rules, config |

### State Management Best Practices

1. **Establish Clear Ownership**: Define which agents own which state keys
2. **Use Consistent Prefixes**: Namespace by lifecycle and domain
3. **Prefer Explicit Data Passing**: Use function parameters over implicit state
4. **Implement Cleanup Steps**: Remove temporary state after use

## Multi-Agent Architecture Patterns

### The Three Foundational Patterns

#### 1. Coordinator Pattern (Hub-and-Spoke)
Central routing and delegation to specialists:

```python
triage_agent = Agent(
    name="TriageAgent",
    model="gemini-2.5-flash",  # Fast, cheap model for routing
    instruction="""Route requests to appropriate specialists:
    - Order queries → CustomerSupportAgent
    - Technical issues → TechnicalAgent""",
    sub_agents=[customer_support, technical_agent]
)
```

**Key Insight**: The `description` field is functional—it drives routing decisions.

#### 2. Pipeline Pattern (Assembly Line)
Strict, sequential execution of steps:

```python
from google.adk.agents import SequentialAgent

technical_issue_agent = SequentialAgent(
    name="TechnicalIssueAgent",
    description="Structured diagnostic process",
    sub_agents=[
        gather_info_agent,      # Step 1
        warranty_checker_agent,  # Step 2
        provide_steps_agent     # Step 3
    ]
)

# State flows between steps
gather_info_agent = Agent(
    name="GatherDeviceInfoAgent",
    output_key="device_info"  # Writes to state
)

provide_steps_agent = Agent(
    name="ProvideStepsAgent",
    instruction="Based on {device_info} and {warranty_status}..."
    # Reads from state using {key} syntax
)
```

#### 3. Specialist Consultation Pattern
Quick expert consultation without full handoff:

```python
from google.adk.tools import AgentTool

# Wrap specialist as callable tool
eligibility_tool = AgentTool(customer_support_assistant)

triage_agent = Agent(
    tools=[eligibility_tool],
    instruction="Use CustomerSupportAssistant tool to verify eligibility before transferring"
)
```

### Workflow Agents

#### SequentialAgent
Executes sub-agents in linear sequence:
```python
film_team = SequentialAgent(
    sub_agents=[research, write, review]
)
```

#### LoopAgent
Repeats sequence until exit condition:
```python
from google.adk.agents import LoopAgent
from google.adk.tools import exit_loop

writers_room = LoopAgent(
    sub_agents=[researcher, writer, critic],
    max_iterations=5
)

critic = Agent(
    tools=[exit_loop],
    instruction="If satisfactory, use exit_loop tool"
)
```

#### ParallelAgent
Executes sub-agents concurrently:
```python
from google.adk.agents import ParallelAgent

preproduction = ParallelAgent(
    sub_agents=[budget_analyst, casting_director]
)
```

## Multimodal Capabilities

### Processing Images/Audio/Video
```python
async def analyze_damage(tool_context: ToolContext) -> dict:
    """Analyzes damage from product images."""
    assessment = {
        "damage_severity": "moderate",
        "warranty_coverage": "Covered"
    }
    
    # Save report as artifact
    report_artifact = types.Part.from_bytes(
        data=json.dumps(assessment).encode(),
        mime_type="application/json"
    )
    
    await tool_context.save_artifact(
        filename="user:damage_report.json",
        artifact=report_artifact
    )
    
    return assessment
```

### Live Streaming Support
```python
from google.adk import RunConfig, StreamingMode

run_config = RunConfig(
    streaming_mode=StreamingMode.BIDI,
    response_modalities=["AUDIO"],
    speech_config=types.SpeechConfig(...)
)
```

## Key Implementation Considerations

### The Multiplication Effect
Costs and latency compound with each hop:
- Single Agent: $0.002, 2 seconds
- Triage → Specialist: $0.006, 5 seconds
- Triage → Specialist → Consultant: $0.012, 8 seconds

**Mitigation Strategies:**
- Use cheaper models for routing (Flash for triage, Pro for reasoning)
- Replace LLM hops with deterministic rules where possible
- Question every agent-to-agent hop: "Is this essential?"

### Debugging and Monitoring
- Implement correlation IDs from day one
- Use the Dev UI for visual debugging
- Monitor delegation patterns and loops
- Set max delegation depth to prevent infinite loops

## Decision Framework

### When to Build Multi-Agent Systems

✅ **Build Multi-Agent When:**
- Clear skill separation (Technical vs Billing vs Legal)
- Deterministic workflows (document processing pipelines)
- Multiple distinct capabilities requiring routing logic
- Different teams own different domains

❌ **Stay Single-Agent When:**
- Skills overlap significantly
- Workflow is highly dynamic
- Single core capability
- Better prompting could solve it

### Architecture Selection Matrix

| Scenario | Pattern | Why |
|----------|---------|-----|
| Help desk with departments | Coordinator | Route to domain experts |
| Invoice processing | Pipeline | Fixed sequential steps |
| Order validation | Consultation | Quick check without handoff |
| Complex returns | Coordinator + Consultation | Route and validate |
| Data ETL | Pipeline | Deterministic flow |

## Development Tips and Best Practices

### Start Simple, Grow Naturally
1. Begin with basic conversation
2. Add tools for data access
3. Introduce state for memory
4. Expand to multimodal when needed
5. Add multi-agent complexity only when necessary

### Common Patterns

**Shopping Cart with State:**
```python
cart = tool_context.state.get("user:cart", {"items": []})
cart["items"].append(item)
tool_context.state["user:cart"] = cart
```

**Document Management:**
```python
# Save important documents
await tool_context.save_artifact(
    filename="user:receipt_12345.pdf",
    artifact=receipt_data
)

# List user documents
artifacts = await tool_context.list_artifacts()
user_docs = [f for f in artifacts if f.startswith("user:")]
```

**Error Handling:**
```python
# ADK converts errors to information
if not order_data:
    return {"error": "Order not found"}
# Agent can reason about the error
```

### Anti-Patterns to Avoid

1. **The Kitchen Sink Coordinator**: Making one agent route everything
2. **The State Soup**: No clear ownership or naming conventions
3. **The Model Maximalist**: Using expensive models for simple routing
4. **The Silent Failure**: No correlation IDs or tracing
5. **The Infinite Loop**: No max delegation depth
6. **The Over-Engineer**: Multi-agent for single-domain problems

## Quick Debugging Checklist

- [ ] Use `adk web --reload_agents` for live development
- [ ] Check state scopes - ensure correct prefixes
- [ ] Verify tool vs subagent decisions
- [ ] Implement correlation IDs across all hops
- [ ] Define state ownership clearly
- [ ] Optimize model sizing (cheap for routing, powerful for reasoning)
- [ ] Set timeout policies for long-running operations
- [ ] Create fallback paths when agents fail

## Key Principles to Remember

1. **ADK handles the complexity** - focus on business logic, not infrastructure
2. **Start with 7 lines, add incrementally** - complexity should be earned
3. **Choose the right tool for the job** - tools for deterministic ops, agents for reasoning
4. **The goal is the simplest system that works** - multi-agent is powerful but not always necessary
5. **Every additional agent multiplies operational overhead** - make sure it's worth it

---

*Remember: ADK's power is in its simplicity. Start small, iterate based on real needs, and let the framework handle the infrastructure complexity while you focus on creating value.*
