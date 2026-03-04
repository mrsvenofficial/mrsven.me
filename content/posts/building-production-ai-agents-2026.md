---
title: "Building Production AI Agents: A Practical Guide to LangChain, CrewAI, and AutoGen"
date: 2026-03-04T19:19:00Z
description: "How to build reliable, scalable AI agents in 2026 using LangChain, CrewAI, and AutoGen with real code examples and production patterns"
tags: ["AI", "Agents", "LangChain", "CrewAI", "AutoGen", "Production", "Code"]
draft: false
---

Last week a developer friend showed me his AI agent project. It was impressive - five agents collaborating to research, analyze, and summarize news stories. But when I asked about error handling, monitoring, and deployment, he stared at me blank.

"This is just a prototype," he said. "I'll figure out production later."

I see this pattern constantly. Developers build cool demos with LangChain, CrewAI, or AutoGen that work 90% of the time. But that 10% failure rate compounds across multi-step workflows, API calls, and external integrations. In production, a 90% reliability rate is unacceptable.

I want to walk through how to build AI agents that actually work in production. Not demos. Not prototypes. Systems that handle errors, recover gracefully, and scale reliably.

## The Framework Landscape in 2026

Three frameworks dominate the AI agent development landscape: LangChain with LangGraph, CrewAI, and AutoGen. Each has strengths, weaknesses, and ideal use cases.

### LangChain and LangGraph

LangChain is the most mature framework. It provides modular components for chaining LLM operations, managing memory, and connecting to tools. LangGraph, its extension, excels at structured, stateful workflows.

Best for: Single-agent applications, copilots, RAG systems, and workflows with clear state transitions.

### CrewAI

CrewAI focuses on multi-agent orchestration. You define roles, tasks, and processes. Agents collaborate sequentially or hierarchically to complete complex workflows.

Best for: Team-like agent systems, complex research workflows, and use cases requiring specialized agent roles.

### AutoGen

AutoGen, from Microsoft, builds multi-agent conversational systems. It has a layered architecture with Core for distributed networks, AgentChat for easy setups, and tools like AutoGen Studio for no-code configuration.

Best for: Advanced multi-agent systems, code generation workflows, and enterprise orchestration at scale.

Choose based on your needs. LangChain for prototyping single agents, CrewAI for task teams, AutoGen for advanced multi-agent scalability.

## Production Architecture Patterns

Before writing code, understand the architecture patterns that make agents reliable in production.

### The Retry Pattern

LLM APIs fail. Network requests timeout. Rate limits hit. You need retry logic with exponential backoff.

### The Circuit Breaker Pattern

When an external service or API fails repeatedly, stop trying. Let it cool down before retrying. This prevents cascading failures.

### The Human-in-the-Loop Pattern

For critical decisions or high-value actions, route to human review before execution. The AI recommends, the human approves.

### The Monitoring Pattern

Track every decision, every tool call, every failure. You cannot debug what you cannot measure.

### The State Management Pattern

Multi-step workflows need durable state. If a process crashes halfway through, it should resume, not restart.

Let me show you how to implement these patterns with each framework.

## LangChain: Production Single Agents

LangChain shines for single-agent applications with clear workflows. Here is a production-ready implementation.

### Basic Agent Setup

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain.tools import Tool
import requests
from datetime import datetime
import logging
from tenacity import retry, stop_after_attempt, wait_exponential
import json

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

# Initialize LLM with production settings
llm = ChatOpenAI(
    model="gpt-4o",
    temperature=0,
    max_retries=3,
    timeout=30
)

# Define tools with error handling
@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
def search_web(query: str) -> str:
    """Search the web for information."""
    logger.info(f"Searching web for: {query}")

    try:
        # Use your search API of choice (Google, Bing, Perplexity, etc.)
        response = requests.post(
            "https://api.search-service.com/search",
            json={"query": query, "num_results": 5},
            timeout=10
        )
        response.raise_for_status()

        results = response.json()
        logger.info(f"Found {len(results.get('results', []))} results")

        return json.dumps(results)

    except Exception as e:
        logger.error(f"Web search failed: {e}")
        raise

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
def fetch_url(url: str) -> str:
    """Fetch content from a URL."""
    logger.info(f"Fetching URL: {url}")

    try:
        response = requests.get(url, timeout=15)
        response.raise_for_status()

        # Limit content size to prevent token overflow
        content = response.text[:50000]
        logger.info(f"Fetched {len(content)} characters")

        return content

    except Exception as e:
        logger.error(f"URL fetch failed: {e}")
        raise

# Create tool list
tools = [
    Tool(
        name="search_web",
        func=search_web,
        description="Search the web for current information"
    ),
    Tool(
        name="fetch_url",
        func=fetch_url,
        description="Fetch and read the content of a webpage"
    )
]

# Create prompt template with guardrails
prompt = ChatPromptTemplate.from_messages([
    ("system", """You are a research assistant that helps gather and synthesize information.

Guidelines:
- Always cite your sources
- If information is unclear or missing, acknowledge it
- Verify facts across multiple sources when possible
- Flag information that seems unreliable or outdated
- Structure your output clearly with headings and bullet points

When in doubt, ask for clarification rather than guessing."""),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}")
])

# Create agent
agent = create_tool_calling_agent(llm, tools, prompt)

# Create executor with production settings
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,
    handle_parsing_errors=True,  # Gracefully handle output parsing errors
    max_iterations=10,  # Prevent infinite loops
    return_intermediate_steps=True  # Track reasoning steps
)

# Wrap with monitoring and state management
class ProductionAgent:
    def __init__(self, executor, state_file="agent_state.json"):
        self.executor = executor
        self.state_file = state_file
        self.metrics = {
            "total_invocations": 0,
            "successful_invocations": 0,
            "failed_invocations": 0,
            "total_tokens": 0,
            "total_cost": 0
        }

    def load_state(self):
        try:
            with open(self.state_file, 'r') as f:
                return json.load(f)
        except FileNotFoundError:
            return {}

    def save_state(self, state):
        with open(self.state_file, 'w') as f:
            json.dump(state, f, indent=2)

    def invoke(self, input_text):
        self.metrics["total_invocations"] += 1
        start_time = datetime.now()

        logger.info(f"Invoking agent with input: {input_text[:100]}...")

        try:
            # Execute agent
            result = self.executor.invoke({"input": input_text})

            # Calculate metrics
            duration = (datetime.now() - start_time).total_seconds()

            # Save execution state
            execution_state = {
                "timestamp": datetime.now().isoformat(),
                "input": input_text,
                "output": result.get("output"),
                "intermediate_steps": [
                    {
                        "tool": step[0].tool,
                        "input": step[0].tool_input,
                        "output": str(step[1])[:500]
                    }
                    for step in result.get("intermediate_steps", [])
                ],
                "duration_seconds": duration,
                "status": "success"
            }

            self.metrics["successful_invocations"] += 1

            logger.info(f"Agent execution completed in {duration:.2f}s")

            return execution_state

        except Exception as e:
            self.metrics["failed_invocations"] += 1
            logger.error(f"Agent execution failed: {e}", exc_info=True)

            # Save failure state
            failure_state = {
                "timestamp": datetime.now().isoformat(),
                "input": input_text,
                "error": str(e),
                "status": "failed"
            }

            raise

    def get_metrics(self):
        success_rate = (
            self.metrics["successful_invocations"] /
            self.metrics["total_invocations"] * 100
            if self.metrics["total_invocations"] > 0 else 0
        )

        return {
            **self.metrics,
            "success_rate_percent": round(success_rate, 2)
        }

# Usage
production_agent = ProductionAgent(agent_executor)

try:
    result = production_agent.invoke(
        "Research the latest developments in agentic AI for March 2026 and provide a summary with sources."
    )

    print("Result:", result["output"])
    print("Intermediate steps:", result["intermediate_steps"])

except Exception as e:
    print(f"Execution failed: {e}")

# Check metrics
print("Agent metrics:", production_agent.get_metrics())
```

### LangGraph for Stateful Workflows

LangGraph shines for complex, multi-step workflows where state management matters.

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated, Sequence
import operator

# Define workflow state
class WorkflowState(TypedDict):
    query: str
    search_results: list
    selected_sources: list
    synthesized_content: str
    citations: list
    quality_score: float
    iteration_count: Annotated[int, operator.add]

# Define nodes
def search_node(state: WorkflowState) -> WorkflowState:
    """Search for information."""
    logger.info("Executing search node")

    # Use the agent to search
    result = agent_executor.invoke({
        "input": f"Search for: {state['query']}"
    })

    return {
        "search_results": [result["output"]],
        "iteration_count": 1
    }

def select_sources_node(state: WorkflowState) -> WorkflowState:
    """Select the best sources from search results."""
    logger.info("Executing source selection node")

    prompt = f"""
    Review these search results and select the 3 most relevant and reliable sources:

    {state['search_results']}

    Return a JSON array with:
    - source: brief description of the source
    - relevance: score from 1-10
    - reliability: score from 1-10
    - key_points: 2-3 main points from this source
    """

    result = llm.invoke(prompt)

    # Parse the JSON response
    import json
    selected_sources = json.loads(result.content)

    return {"selected_sources": selected_sources}

def synthesize_node(state: WorkflowState) -> WorkflowState:
    """Synthesize information from selected sources."""
    logger.info("Executing synthesis node")

    sources_text = "\n\n".join([
        f"Source {i+1}: {source['source']}\nKey Points: {source['key_points']}"
        for i, source in enumerate(state['selected_sources'])
    ])

    prompt = f"""
    Synthesize the following information about "{state['query']}":

    {sources_text}

    Provide:
    1. A comprehensive summary
    2. Key insights and trends
    3. Citations in [Source N] format
    4. Areas where information is conflicting or unclear
    """

    result = llm.invoke(prompt)

    return {
        "synthesized_content": result.content,
        "iteration_count": 1
    }

def quality_check_node(state: WorkflowState) -> WorkflowState:
    """Check the quality of synthesized content."""
    logger.info("Executing quality check node")

    prompt = f"""
    Evaluate the quality of this synthesized content:

    {state['synthesized_content']}

    Rate on a scale of 1-10 for:
    1. Completeness
    2. Accuracy (based on sources)
    3. Clarity
    4. Citations

    Return just the average score as a number.
    """

    result = llm.invoke(prompt)

    try:
        quality_score = float(result.content.strip())
    except:
        quality_score = 7.0  # Default score

    logger.info(f"Quality score: {quality_score}")

    return {"quality_score": quality_score}

# Build the graph
workflow = StateGraph(WorkflowState)

# Add nodes
workflow.add_node("search", search_node)
workflow.add_node("select_sources", select_sources_node)
workflow.add_node("synthesize", synthesize_node)
workflow.add_node("quality_check", quality_check_node)

# Add edges
workflow.set_entry_point("search")
workflow.add_edge("search", "select_sources")
workflow.add_edge("select_sources", "synthesize")
workflow.add_edge("synthesize", "quality_check")

# Add conditional edge for quality-based iteration
def should_refine(state: WorkflowState) -> str:
    """Decide whether to refine based on quality score."""
    if state["quality_score"] < 8 and state["iteration_count"] < 3:
        logger.info(f"Quality score {state['quality_score']} below threshold, refining...")
        return "synthesize"
    else:
        return END

workflow.add_conditional_edges(
    "quality_check",
    should_refine,
    {"synthesize": "synthesize", END: END}
)

# Compile the workflow
app = workflow.compile()

# Execute
initial_state = {
    "query": "AI automation production deployments 2026",
    "search_results": [],
    "selected_sources": [],
    "synthesized_content": "",
    "citations": [],
    "quality_score": 0.0,
    "iteration_count": 0
}

# Run the workflow
final_state = app.invoke(initial_state)

print("Final content:")
print(final_state["synthesized_content"])
print(f"\nQuality score: {final_state['quality_score']}")
print(f"Iterations: {final_state['iteration_count']}")
```

## CrewAI: Multi-Agent Collaboration

CrewAI excels when you need specialized agents working together. Here is a production-ready implementation for a research and analysis workflow.

```python
from crewai import Agent, Task, Crew, Process
from langchain_openai import ChatOpenAI
import logging
from datetime import datetime
import json

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

# Initialize LLM
llm = ChatOpenAI(
    model="gpt-4o",
    temperature=0.1,
    max_retries=3,
    timeout=30
)

# Define specialized agents
researcher = Agent(
    role="Senior Research Analyst",
    goal="Uncover cutting-edge developments in AI and automation",
    backstory="""You are a seasoned research analyst with 15 years of experience
    tracking AI industry trends. You have a knack for finding reliable sources
    and extracting actionable insights from complex information.""",
    verbose=True,
    allow_delegation=False,
    llm=llm,
    max_iter=3,  # Prevent infinite loops
    max_execution_time=300,  # 5 minute timeout
    tools=[]  # Add tools as needed
)

analyst = Agent(
    role="Data Analyst",
    goal="Analyze research findings and identify patterns and insights",
    backstory="""You are a data analyst who specializes in finding the signal
    in the noise. You excel at quantitative analysis, trend identification, and
    translating data into clear recommendations.""",
    verbose=True,
    allow_delegation=False,
    llm=llm,
    max_iter=3,
    max_execution_time=300
)

writer = Agent(
    role="Technical Writer",
    goal="Synthesize research and analysis into clear, actionable content",
    backstory="""You are a technical writer who bridges the gap between complex
    technical concepts and practical implementation guidance. You write clearly,
    concisely, and with the reader's needs in mind.""",
    verbose=True,
    allow_delegation=False,
    llm=llm,
    max_iter=2,
    max_execution_time=180
)

quality_checker = Agent(
    role="Quality Assurance Specialist",
    goal="Ensure all content meets accuracy, clarity, and sourcing standards",
    backstory="""You are a meticulous QA specialist who catches errors, verifies
    claims, and ensures content meets high editorial standards. Nothing gets past
    your review without thorough verification.""",
    verbose=True,
    allow_delegation=False,
    llm=llm,
    max_iter=2,
    max_execution_time=120
)

# Define tasks
research_task = Task(
    description="""Research the latest developments in AI agent production
    deployments for March 2026. Focus on:

    1. Real-world case studies with specific metrics (ROI, efficiency gains, etc.)
    2. Tools and frameworks being used in production
    3. Common failure patterns and how companies overcame them
    4. Emerging best practices for reliability and scalability

    Gather information from at least 5 diverse sources. Verify facts across
    multiple sources when possible. Note any conflicting information.

    Provide a structured report with citations in [Source N] format.""",
    agent=researcher,
    expected_output="A comprehensive research report with verified facts and citations"
)

analysis_task = Task(
    description="""Analyze the research findings and identify:

    1. Key patterns and trends across different industries
    2. Quantifiable metrics (average ROI, common timeframes, etc.)
    3. Success factors that correlate with positive outcomes
    4. Risk factors that correlate with failures
    5. Gaps or areas needing further research

    Provide specific, data-driven insights with supporting evidence.""",
    agent=analyst,
    expected_output="A data-driven analysis identifying patterns, metrics, and insights",
    context=[research_task]  # Pass research results to analysis
)

writing_task = Task(
    description="""Write a comprehensive article based on the research and analysis.
    Structure the article as:

    1. Introduction (why this matters now)
    2. Current state of AI agents in production
    3. Real-world case studies with specific examples
    4. Key insights and patterns
    5. Common failure patterns and how to avoid them
    6. Implementation guide with actionable recommendations
    7. Conclusion

    Requirements:
    - Include code examples where relevant
    - Cite sources properly throughout
    - Maintain a professional, authoritative tone
    - Make it practical and actionable
    - Avoid AI cliches and fluff

    Length: 2000-3000 words""",
    agent=writer,
    expected_output="A polished, well-structured article ready for publication",
    context=[research_task, analysis_task]
)

quality_check_task = Task(
    description="""Review the article for:

    1. Factual accuracy (verify claims against research sources)
    2. Proper sourcing and citation format
    3. Clarity and readability
    4. Absence of AI writing patterns (no "delve", "underscore", etc.)
    5. Practical value and actionability

    If issues are found:
    - List them clearly with specific examples
    - Suggest specific fixes
    - Flag anything that requires writer revision

    If content passes:
    - Confirm it meets quality standards
    - Note any minor suggestions for improvement""",
    agent=quality_checker,
    expected_output="A detailed quality review with either approval or specific revisions needed",
    context=[writing_task]
)

# Create crew with production settings
crew = Crew(
    agents=[researcher, analyst, writer, quality_checker],
    tasks=[research_task, analysis_task, writing_task, quality_check_task],
    process=Process.hierarchical,  # Sequential execution with manager
    manager_llm=llm,  # LLM for task coordination
    verbose=True,
    memory=True,  # Enable memory for context retention
    cache=True,  # Cache results to save tokens and time
    max_rpm=100,  # Rate limiting for API calls
    share_crew=False  # Don't expose crew as API
)

# Wrap with monitoring and error handling
class ProductionCrew:
    def __init__(self, crew, log_file="crew_execution.log"):
        self.crew = crew
        self.log_file = log_file
        self.execution_history = []

    def execute(self, inputs):
        execution_id = datetime.now().strftime("%Y%m%d_%H%M%S")
        start_time = datetime.now()

        logger.info(f"Starting crew execution {execution_id}")
        logger.info(f"Inputs: {json.dumps(inputs, indent=2)}")

        try:
            # Execute crew
            result = self.crew.kickoff(inputs=inputs)

            # Calculate metrics
            duration = (datetime.now() - start_time).total_seconds()

            execution_record = {
                "execution_id": execution_id,
                "timestamp": datetime.now().isoformat(),
                "inputs": inputs,
                "result": str(result),
                "duration_seconds": duration,
                "status": "success"
            }

            self.execution_history.append(execution_record)

            logger.info(f"Crew execution completed in {duration:.2f}s")

            return execution_record

        except Exception as e:
            duration = (datetime.now() - start_time).total_seconds()

            logger.error(f"Crew execution failed after {duration:.2f}s: {e}", exc_info=True)

            execution_record = {
                "execution_id": execution_id,
                "timestamp": datetime.now().isoformat(),
                "inputs": inputs,
                "error": str(e),
                "duration_seconds": duration,
                "status": "failed"
            }

            self.execution_history.append(execution_record)

            # Save execution history
            self._save_history()

            raise

    def _save_history(self):
        with open(self.log_file, 'a') as f:
            for record in self.execution_history:
                f.write(json.dumps(record) + "\n")

    def get_stats(self):
        total = len(self.execution_history)
        successful = sum(1 for r in self.execution_history if r["status"] == "success")
        failed = total - successful
        avg_duration = sum(r["duration_seconds"] for r in self.execution_history) / total if total > 0 else 0

        return {
            "total_executions": total,
            "successful": successful,
            "failed": failed,
            "success_rate": f"{(successful/total*100):.1f}%" if total > 0 else "N/A",
            "average_duration_seconds": round(avg_duration, 2)
        }

# Usage
production_crew = ProductionCrew(crew)

try:
    result = production_crew.execute({
        "topic": "AI automation production deployments",
        "date_range": "March 2026",
        "focus": "case studies and implementation patterns"
    })

    print("Execution result:", result)

except Exception as e:
    print(f"Execution failed: {e}")

# Check stats
print("Crew stats:", production_crew.get_stats())
```

## AutoGen: Advanced Multi-Agent Systems

AutoGen provides sophisticated multi-agent orchestration with built-in conversation patterns and state management.

```python
from autogen import AssistantAgent, UserProxyAgent, GroupChat, GroupChatManager
from autogen.coding import DockerCommandLineCodeExecutor
import os
import logging
from datetime import datetime
import json

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

# Define config for LLMs
config_list = [
    {
        "model": "gpt-4o",
        "api_key": os.environ.get("OPENAI_API_KEY"),
        "temperature": 0.1,
        "max_tokens": 2000
    }
]

# Create code executor (for agents that need to run code)
code_executor = DockerCommandLineCodeExecutor(
    image="python:3.11-slim",
    timeout=60
)

# Create specialized agents
researcher = AssistantAgent(
    name="Researcher",
    llm_config={"config_list": config_list},
    system_message="""You are a research specialist who:
    - Finds and verifies information from reliable sources
    - Cites sources properly
    - Identifies conflicting information
    - Provides structured, evidence-based outputs
    - Focuses on facts over speculation"""
)

analyst = AssistantAgent(
    name="Analyst",
    llm_config={"config_list": config_list},
    system_message="""You are a data analyst who:
    - Identifies patterns and trends
    - Calculates metrics and statistics
    - Provides data-driven insights
    - Validates assumptions
    - Presents findings clearly"""
)

engineer = AssistantAgent(
    name="Engineer",
    llm_config={"config_list": config_list},
    system_message="""You are a software engineer who:
    - Writes clean, production-ready code
    - Implements error handling and logging
    - Follows best practices and patterns
    - Tests code before recommending it
    - Provides implementation guidance""",
    code_execution_config={
        "work_dir": "coding",
        "use_docker": True,
        "executor": code_executor
    }
)

reviewer = AssistantAgent(
    name="Reviewer",
    llm_config={"config_list": config_list},
    system_message="""You are a code reviewer who:
    - Reviews code for correctness and best practices
    - Identifies security vulnerabilities
    - Checks for error handling edge cases
    - Provides constructive feedback
    - Ensures code is production-ready"""
)

# Create user proxy to initiate conversations
user_proxy = UserProxyAgent(
    name="User",
    code_execution_config=False,  # User doesn't execute code
    human_input_mode="NEVER",  # Fully autonomous
    max_consecutive_auto_reply=10
)

# Create group chat
groupchat = GroupChat(
    agents=[researcher, analyst, engineer, reviewer, user_proxy],
    messages=[],
    max_round=20,  # Prevent infinite loops
    speaker_selection_method="round_robin"
)

# Create manager with state persistence
manager = GroupChatManager(
    groupchat=groupchat,
    llm_config={"config_list": config_list},
    code_execution_config=False,
    max_consecutive_auto_reply=10
)

# Wrap with production features
class ProductionAutoGenOrchestrator:
    def __init__(self, manager, state_file="autogen_state.json"):
        self.manager = manager
        self.state_file = state_file
        self.conversation_history = []
        self.metrics = {
            "total_conversations": 0,
            "successful_conversations": 0,
            "failed_conversations": 0,
            "total_messages": 0,
            "average_turns_per_conversation": 0
        }

    def load_state(self):
        try:
            with open(self.state_file, 'r') as f:
                return json.load(f)
        except FileNotFoundError:
            return {}

    def save_state(self, state):
        with open(self.state_file, 'w') as f:
            json.dump(state, f, indent=2)

    def orchestrate(self, initial_message):
        conversation_id = datetime.now().strftime("%Y%m%d_%H%M%S")
        start_time = datetime.now()

        logger.info(f"Starting conversation {conversation_id}")
        logger.info(f"Initial message: {initial_message[:200]}...")

        self.metrics["total_conversations"] += 1

        try:
            # Initiate conversation
            result = user_proxy.initiate_chat(
                manager,
                message=initial_message,
                clear_history=True
            )

            # Calculate metrics
            duration = (datetime.now() - start_time).total_seconds()
            num_messages = len(manager.chat_history)

            conversation_record = {
                "conversation_id": conversation_id,
                "timestamp": datetime.now().isoformat(),
                "initial_message": initial_message,
                "num_messages": num_messages,
                "duration_seconds": duration,
                "status": "success",
                "summary": result.summary if hasattr(result, 'summary') else str(result)[-500:]
            }

            self.conversation_history.append(conversation_record)
            self.metrics["successful_conversations"] += 1
            self.metrics["total_messages"] += num_messages

            logger.info(f"Conversation completed in {duration:.2f}s with {num_messages} messages")

            # Save state
            self.save_state({
                "conversation_history": self.conversation_history[-10:],  # Last 10 conversations
                "metrics": self.metrics
            })

            return conversation_record

        except Exception as e:
            duration = (datetime.now() - start_time).total_seconds()

            logger.error(f"Conversation failed after {duration:.2f}s: {e}", exc_info=True)

            conversation_record = {
                "conversation_id": conversation_id,
                "timestamp": datetime.now().isoformat(),
                "initial_message": initial_message,
                "error": str(e),
                "duration_seconds": duration,
                "status": "failed"
            }

            self.conversation_history.append(conversation_record)
            self.metrics["failed_conversations"] += 1

            raise

    def get_metrics(self):
        total = self.metrics["total_conversations"]
        self.metrics["average_turns_per_conversation"] = (
            self.metrics["total_messages"] / total if total > 0 else 0
        )

        success_rate = (
            self.metrics["successful_conversations"] / total * 100
            if total > 0 else 0
        )

        return {
            **self.metrics,
            "success_rate_percent": round(success_rate, 2)
        }

# Usage
orchestrator = ProductionAutoGenOrchestrator(manager)

try:
    result = orchestrator.orchestrate(
        """We need to build a production-ready AI agent for customer support.

Requirements:
1. Handle common customer inquiries (password resets, account issues, billing)
2. Integrate with existing CRM system
3. Route complex issues to human agents
4. Track metrics and performance
5. Include error handling and monitoring

Please:
1. Research best practices for AI customer support agents
2. Analyze the technical requirements
3. Implement a prototype using LangChain with proper error handling
4. Review the code for production readiness
5. Provide deployment and monitoring recommendations"""
    )

    print("Conversation result:", result)

except Exception as e:
    print(f"Orchestration failed: {e}")

# Check metrics
print("Orchestrator metrics:", orchestrator.get_metrics())
```

## Production Deployment Patterns

Building agents is step one. Deploying them reliably is step two.

### Docker Containerization

```dockerfile
# Dockerfile for LangChain agent
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY src/ ./src/

# Set environment variables
ENV PYTHONUNBUFFERED=1
ENV LOG_LEVEL=INFO

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8000/health')"

# Run the application
CMD ["python", "src/main.py"]
```

### Docker Compose for Multi-Service Setup

```yaml
# docker-compose.yml
version: '3.8'

services:
  agent:
    build: .
    ports:
      - "8000:8000"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - DATABASE_URL=postgresql://user:pass@db:5432/agents
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
    volumes:
      - ./logs:/app/logs
      - ./state:/app/state
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=agents
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
    restart: unless-stopped

volumes:
  postgres_data:
  prometheus_data:
  grafana_data:
```

### Monitoring with Prometheus

```python
# monitoring.py
from prometheus_client import Counter, Histogram, Gauge, start_http_server
import time

# Define metrics
invocation_counter = Counter('agent_invocations_total', 'Total agent invocations')
invocation_duration = Histogram('agent_invocation_duration_seconds', 'Agent invocation duration')
error_counter = Counter('agent_errors_total', 'Total agent errors', ['error_type'])
active_invocations = Gauge('agent_active_invocations', 'Currently active invocations')

# Metrics decorator
def track_invocation(func):
    def wrapper(*args, **kwargs):
        active_invocations.inc()
        start_time = time.time()

        try:
            result = func(*args, **kwargs)
            invocation_counter.inc()
            return result

        except Exception as e:
            error_type = type(e).__name__
            error_counter.labels(error_type=error_type).inc()
            raise

        finally:
            invocation_duration.observe(time.time() - start_time)
            active_invocations.dec()

    return wrapper

# Start metrics server
start_http_server(8001)
```

### State Persistence with Database

```python
# persistence.py
import psycopg2
from psycopg2.extras import Json
from datetime import datetime
import json

class AgentStateStore:
    def __init__(self, connection_string):
        self.conn = psycopg2.connect(connection_string)
        self._create_tables()

    def _create_tables(self):
        with self.conn.cursor() as cur:
            cur.execute("""
                CREATE TABLE IF NOT EXISTS agent_executions (
                    id SERIAL PRIMARY KEY,
                    agent_id VARCHAR(255) NOT NULL,
                    input_data JSONB,
                    output_data JSONB,
                    status VARCHAR(50),
                    error_message TEXT,
                    duration_seconds FLOAT,
                    created_at TIMESTAMP DEFAULT NOW(),
                    completed_at TIMESTAMP
                );

                CREATE TABLE IF NOT EXISTS agent_state (
                    agent_id VARCHAR(255) PRIMARY KEY,
                    state_data JSONB NOT NULL,
                    updated_at TIMESTAMP DEFAULT NOW()
                );

                CREATE INDEX IF NOT EXISTS idx_executions_agent_id
                ON agent_executions(agent_id);
            """)
            self.conn.commit()

    def save_execution(self, agent_id, input_data, output_data, status,
                       error_message=None, duration_seconds=None):
        with self.conn.cursor() as cur:
            cur.execute("""
                INSERT INTO agent_executions
                (agent_id, input_data, output_data, status, error_message, duration_seconds, completed_at)
                VALUES (%s, %s, %s, %s, %s, %s, NOW())
                RETURNING id
            """, (
                agent_id,
                Json(input_data),
                Json(output_data),
                status,
                error_message,
                duration_seconds
            ))

            execution_id = cur.fetchone()[0]
            self.conn.commit()

            return execution_id

    def get_execution_history(self, agent_id, limit=100):
        with self.conn.cursor() as cur:
            cur.execute("""
                SELECT * FROM agent_executions
                WHERE agent_id = %s
                ORDER BY created_at DESC
                LIMIT %s
            """, (agent_id, limit))

            columns = [desc[0] for desc in cur.description]
            return [dict(zip(columns, row)) for row in cur.fetchall()]

    def save_state(self, agent_id, state_data):
        with self.conn.cursor() as cur:
            cur.execute("""
                INSERT INTO agent_state (agent_id, state_data, updated_at)
                VALUES (%s, %s, NOW())
                ON CONFLICT (agent_id)
                DO UPDATE SET state_data = EXCLUDED.state_data, updated_at = NOW()
            """, (agent_id, Json(state_data)))

            self.conn.commit()

    def load_state(self, agent_id):
        with self.conn.cursor() as cur:
            cur.execute("""
                SELECT state_data FROM agent_state WHERE agent_id = %s
            """, (agent_id,))

            row = cur.fetchone()
            if row:
                return row[0]
            return None
```

## Choosing the Right Framework

Based on production deployments, here is when to use each framework.

### Use LangChain When:
- Building single-agent applications (chatbots, copilots, assistants)
- Need stateful workflows with clear transitions
- Want the most mature ecosystem and community support
- Integrating with RAG systems or vector databases
- Need fine-grained control over agent behavior

### Use CrewAI When:
- Building multi-agent systems with specialized roles
- Want simpler setup than AutoGen
- Need sequential or hierarchical task delegation
- Building team-like collaboration workflows
- Want clear role separation (researcher, analyst, writer, etc.)

### Use AutoGen When:
- Building complex multi-agent systems at scale
- Need conversational patterns between agents
- Want enterprise-grade orchestration
- Building code generation or review workflows
- Need distributed agent deployment

## Common Production Pitfalls

### 1. No Error Handling

Agents fail silently or crash without recovery. Always implement retry logic, circuit breakers, and graceful degradation.

### 2. Infinite Loops

Multi-agent conversations run forever without termination conditions. Set max iterations, time limits, and convergence criteria.

### 3. State Loss

If a process crashes, all progress is lost. Persist state after each step and enable recovery.

### 4. Cost Blindness

Unmonitored API calls burn through budget. Track token usage, cache results, use cheaper models for simple tasks.

### 5. No Observability

You cannot debug what you cannot measure. Log every decision, track every metric, monitor every failure.

### 6. Missing Guardrails

Agents take actions without validation. Implement approval thresholds, safety checks, and human oversight for critical operations.

## The Path Forward

Building production AI agents is not about picking the right framework. It is about thinking like a production engineer.

Design for failure. Monitor everything. Measure ROI. Iterate based on data.

The developers I see succeeding in 2026 are the ones who treat AI agents like any other production system. They write tests. They deploy gradually. They monitor metrics. They fix bugs. They scale based on data, not hype.

Start with a single agent on LangChain. Add error handling and monitoring. Deploy to production. Measure results.

Then expand. Add a second agent in CrewAI. Orchestrate them with AutoGen. Scale to handle volume.

The frameworks are tools. Production engineering is the skill. Focus there, and your agents will work.
