---
title: "I Built an AI Agent That Handles My Daily Grunt Work"
date: 2026-03-09
description: "A practical guide to building autonomous AI agents that actually ship work, not just chat about it."
tags: ["ai", "automation", "agents", "workflow", "productivity"]
---

I'll be honest: I spent months experimenting with AI tools that promised to "automate my workflow" but mostly just automated more conversations. Same chatbot interface, slightly better answers. Still on me to copy-paste, verify, integrate, ship.

Three weeks ago I stopped accepting that as good enough.

## The Breaking Point

My Tuesday looked like this:

1. Morning emails triage (30 min)
2. Check GitHub PRs and update tracking spreadsheet (20 min)
3. Respond to support tickets, copy urgent ones to Slack (25 min)
4. Monitor deployed service health, check logs for alerts (15 min)
5. Weekly metrics report collation (45 min on Friday)
6. Repeat variations of this across the week

None of this is hard work. That's the problem. It's tedious, context-switching work that breaks flow. A 90-minute deep-work session becomes 20 minutes of actual coding sandwiched between notification checks and spreadsheet updates.

I didn't want an AI assistant. I wanted an AI coworker who just handles this stuff and tells me when it's done.

## What I Built

I created three autonomous agents using OpenAI's Assistants API plus some glue code:

**Agent 1: Triage Agent**
- Scans inbox every 2 hours
- Categorizes emails: urgent, respond-later, info-only, spam
- Drafts responses for urgent items
- Posts summary to a dedicated Slack channel

**Agent 2: DevOps Agent**
- Monitors deployed services via API
- Checks logs for errors and anomalies
- Restarts failed services if safe (requires manual approval for databases)
- Posts incidents to Slack with relevant log excerpts

**Agent 3: Reporting Agent**
- Runs every Monday at 9 AM
- Pulls metrics from multiple sources (Stripe, GitHub, Analytics)
- Generates a markdown report with trends and anomalies
- Posts to our team channel

These aren't chatbots. They run on schedules, take actions, and send me human-readable summaries. When something needs my attention, they ping me with context and a clear question. Otherwise, they just work.

## The Architecture

Here's the actual implementation. I'll show you the code and explain why I made these choices.

### 1. The Agent Runtime

I built a simple runtime that manages the agent lifecycle:

```python
# agent_runtime.py
import os
import time
import asyncio
from openai import AsyncOpenAI
from typing import Callable, Dict, Any

class AgentRuntime:
    def __init__(self):
        self.client = AsyncOpenAI(api_key=os.environ['OPENAI_API_KEY'])
        self.agents: Dict[str, Dict] = {}

    def register_agent(self, name: str, instructions: str, tools: list):
        assistant = self.client.beta.assistants.create(
            name=name,
            instructions=instructions,
            tools=tools,
            model="gpt-4o"
        )
        self.agents[name] = {
            'assistant_id': assistant.id,
            'tools': tools
        }

    async def run_agent(self, agent_name: str, input_text: str) -> str:
        agent = self.agents[agent_name]
        thread = self.client.beta.threads.create(
            messages=[{"role": "user", "content": input_text}]
        )

        run = self.client.beta.threads.runs.create(
            thread_id=thread.id,
            assistant_id=agent['assistant_id']
        )

        while True:
            run_status = self.client.beta.threads.runs.retrieve(
                thread_id=thread.id,
                run_id=run.id
            )

            if run_status.status == 'completed':
                messages = self.client.beta.threads.messages.list(thread_id=thread.id)
                return messages.data[0].content[0].text.value
            elif run_status.status == 'requires_action':
                # Handle tool calls
                tool_outputs = []
                for tool_call in run_status.required_action.submit_tool_outputs.tool_calls:
                    # Execute the actual function
                    tool_outputs.append({
                        "tool_call_id": tool_call.id,
                        "output": str(execute_tool(tool_call.function.name, tool_call.function.arguments))
                    })

                run = self.client.beta.threads.runs.submit_tool_outputs(
                    thread_id=thread.id,
                    run_id=run.id,
                    tool_outputs=tool_outputs
                )
            elif run_status.status in ['failed', 'cancelled', 'expired']:
                raise Exception(f"Agent run failed: {run_status.last_error}")

            await asyncio.sleep(1)
```

This runtime handles the async polling that most people skip. The OpenAI Assistants API doesn't give you a callback when a run completes, so you have to poll. Doing this synchronously would block everything, so I made it async and reused the same runtime across agents.

### 2. The Triage Agent Implementation

```python
# triage_agent.py
import imaplib
import email
from email.header import decode_header
from datetime import datetime
import re

def fetch_emails(limit=50) -> list:
    """Fetch recent emails from IMAP."""
    mail = imaplib.IMAP4_SSL(os.environ['IMAP_SERVER'])
    mail.login(os.environ['EMAIL_USER'], os.environ['EMAIL_PASSWORD'])
    mail.select('inbox')

    # Search for recent emails
    status, messages = mail.search(None, f'(UNSEEN)')
    email_ids = messages[0].split()[:limit]

    emails = []
    for e_id in email_ids:
        _, msg_data = mail.fetch(e_id, '(RFC822)')
        raw_email = msg_data[0][1]
        msg = email.message_from_bytes(raw_email)

        subject = decode_header(msg['Subject'])[0][0]
        if isinstance(subject, bytes):
            subject = subject.decode()

        sender = msg['From']
        body = get_email_body(msg)

        emails.append({
            'id': e_id.decode(),
            'subject': subject,
            'sender': sender,
            'body': body[:2000],  # Truncate for token efficiency
            'date': msg['Date']
        })

    mail.close()
    mail.logout()
    return emails

async def run_triage():
    """Run the triage agent."""
    runtime = AgentRuntime()

    runtime.register_agent(
        name="Email Triage",
        instructions="""
You are an email triage assistant. Your job is to categorize incoming emails.

Categories:
1. URGENT: Requires immediate attention (customer issues, outages, time-sensitive requests)
2. RESPOND_LATER: Needs a response but can wait until end of day
3. INFO_ONLY: FYI, newsletters, notifications - no action needed
4. SPAM: Promotional, obvious junk

For each email, output in this JSON format:
{
  "email_id": "...",
  "category": "URGENT|RESPOND_LATER|INFO_ONLY|SPAM",
  "reason": "Brief explanation",
  "draft_response": "Only for URGENT - draft a response, otherwise null"
}

Be conservative with URGENT. Most things are not urgent.
""",
        tools=[{
            "type": "function",
            "function": {
                "name": "send_slack_message",
                "description": "Send a message to a Slack channel",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "channel": {"type": "string"},
                        "message": {"type": "string"}
                    },
                    "required": ["channel", "message"]
                }
            }
        }]
    )

    emails = fetch_emails(limit=30)

    if not emails:
        return

    # Batch process emails
    email_summary = "\n\n---\n\n".join([
        f"ID: {e['id']}\nFrom: {e['sender']}\nSubject: {e['subject']}\nBody: {e['body']}"
        for e in emails
    ])

    result = await runtime.run_agent("Email Triage", email_summary)

    # Parse and act on the result
    try:
        import json
        triage_results = json.loads(result)

        # Post summary to Slack
        urgent = [t for t in triage_results if t['category'] == 'URGENT']
        respond_later = [t for t in triage_results if t['category'] == 'RESPOND_LATER']

        summary = f"*Email Triage - {datetime.now().strftime('%H:%M')}*\n\n"
        summary += f"🔴 Urgent: {len(urgent)}\n"
        summary += f"🟡 Respond Later: {len(respond_later)}\n"
        summary += f"🟢 Info Only: {len([t for t in triage_results if t['category'] == 'INFO_ONLY'])}\n\n"

        if urgent:
            summary += "*Urgent Items:*\n"
            for u in urgent:
                summary += f"- {u['subject'][:50]}...\n"

        await send_slack_message("#email-triage", summary)

    except json.JSONDecodeError:
        await send_slack_message("#email-triage", f"Triage failed to parse results:\n{result}")
```

This agent doesn't just categorize emails, it takes actions. When it finds something urgent, it drafts a response and posts everything to Slack where I can quickly approve or edit.

### 3. Scheduling with Cron

I use a simple scheduler to run agents periodically:

```python
# scheduler.py
import asyncio
from apscheduler.schedulers.asyncio import AsyncIOScheduler

scheduler = AsyncIOScheduler()

async def run_all_triage():
    await run_triage()

async def run_all_monitoring():
    await run_devops_agent()

async def run_all_reports():
    await run_reporting_agent()

# Schedule agents
scheduler.add_job(run_all_triage, 'interval', hours=2)
scheduler.add_job(run_all_monitoring, 'interval', minutes=15)
scheduler.add_job(run_all_reports, 'cron', day_of_week='mon', hour=9)

scheduler.start()

# Keep running
try:
    asyncio.get_event_loop().run_forever()
except KeyboardInterrupt:
    scheduler.shutdown()
```

I deploy this as a small container on my server. It uses minimal resources and just keeps running.

## What I Learned

### 1. Tool Calling Changes Everything

The Assistants API's tool calling feature is what makes this actually work. When the agent needs to send a Slack message or call an API, it doesn't hallucinate JSON or make up function names. It calls the actual function with structured parameters, gets the result, and continues.

Before this, I was prompting models to "output JSON" and parsing that. It worked 80% of the time and failed catastrophically the other 20%. Tool calling is reliable.

### 2. Instructions Matter More Than Prompts

I used to write elaborate prompt engineering for each task. Now I write clear, concise instructions for the agent itself. The model's system prompt is its instructions. Every user message is a task. The tools are its capabilities.

Good agent instructions:
- State the job clearly in one sentence
- Define categories or decisions it can make
- Specify output format precisely (JSON, markdown, etc.)
- Set guardrails ("Be conservative with URGENT")

Bad agent instructions:
- Try to be clever or conversational
- Embed examples of every edge case
- Leave decisions ambiguous
- Expect the model to know your context

### 3. Humans Stay in the Loop

I don't let my agents run wild. Every action that affects production, every message sent to customers, every restart of a service - those require either:
- Automatic safe actions (logging, categorization)
- Human approval (restarts, customer emails)

The DevOps agent can restart web servers automatically. It cannot restart databases without posting to Slack and waiting for approval. I've had it catch and fix a stuck worker process at 3 AM without waking me up. I've also had it flag a database anomaly that I would have missed.

### 4. Start Small, Scale Later

I didn't start with this three-agent system. I started with one agent that just categorized emails. Once I trusted that, I added Slack posting. Then I built the DevOps monitor from scratch with the lessons learned.

The triage agent alone saved me ~2 hours per day. The DevOps agent has already prevented one outage. The reporting agent freed up my Friday afternoons.

## The Results

After three weeks:

- **Time saved:** 10-12 hours per week on routine tasks
- **Missed emails:** 0 (the triage agent caught everything)
- **Incidents detected:** 3 (two minor, one that would have escalated)
- **Stress level:** Way down. I don't check email constantly anymore.

But the real benefit is headspace. When I sit down to code, I actually code. I'm not thinking "I should check those support tickets" or "did that deploy go through?"

## What's Next

I'm working on:

1. **Customer Support Agent**: Trains on our docs and past tickets, handles common queries, escalates when unsure
2. **Content Research Agent**: Monitors competitors and industry news, sends weekly digest
3. **Build Agent**: Runs automated tests, creates PRs for dependency updates

I'm also experimenting with giving agents access to more tools - direct database queries, GitHub API for PR management, Stripe for subscription handling.

## The Important Part

This isn't about replacing humans. It's about giving humans leverage. I still review every customer-facing message. I still make architectural decisions. I still talk to customers and understand their problems.

But I don't do the grunt work anymore. The grunt work is handled. I do the work that actually matters.

## Getting Started

If you want to build your own agents, start here:

1. **Pick one repetitive task** - email triage, log monitoring, report generation
2. **Write a single clear instruction** for what the agent should do
3. **Give it one tool** - a function it can call to take action
4. **Deploy and watch** - see how it performs, iterate
5. **Add more tools gradually** - don't overengineer day one

The OpenAI Assistants API documentation has good examples. The key is understanding that you're building an autonomous worker, not a chatbot. Instructions are its training. Tools are its hands. Runtime is its brain.

You don't need to be an ML engineer. I'm not. I just read the docs, started simple, and iterated.

The technology is there. The question is what you'll build with it.

---

**P.S.** If you build something cool with autonomous agents, I'd love to hear about it. Drop me a message on Twitter [@MrSvenOfficial](https://twitter.com/MrSvenOfficial) and show me what you shipped.
