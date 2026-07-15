---
title: "Building Your Own MCP Server: When Vendor Tools Aren't Enough"
date: 2026-07-08 00:00:00 +0200
image: /assets/img/posts/building-your-own-mcp-server/splash.png
description: |
  PagerDuty and Jira both have MCP servers. Neither knows your infrastructure.
  Here's how to build the custom MCP server that fills the gap between them, so your agent can reason across incidents, logs, and deployments without you manually stitching the answers together.
categories: [AI & ML]
tags: [mcp, ai-agents, aws, incident-response, python]
---

It's 2am. The alarm bells are ringing, CloudWatch is shrieking about `payments-prod`. Everything is on fire and it's your bleary eyed time to shine. Your AI coding agent already has PagerDuty, Jira, and CloudWatch wired up as MCP servers, so you ask it to pull the incident together.

It calls PagerDuty and gets the incident details: severity, alert source, timestamp. It calls Jira and turns up a ticket from a similar incident last month. Neither one knows which CloudWatch log group belongs to `payments-prod`, so you look that up yourself and paste it back in. Same story for the CodePipeline pipeline that shipped the change an hour ago. Three tool calls, three vendors, and you're still the one holding the pieces together. You ask yourself if you're the bottleneck. Kind of, at the very least, your incident response process is.

So how did we get here? Well as you can see, the tools we use to manage incidents are fragmented. Each vendor has built their own MCP server, but none of them know your infrastructure. PagerDuty knows incidents, Jira knows tickets, AWS knows its documentation, but none of them know which CloudWatch log group belongs to which service, which pipeline deployed what, or which team owns which resource.

That knowledge lives in your head. Or in your runbooks. Or scattered across a dozen internal wikis. And that's exactly where a custom MCP server comes in.

## Filling the gap between vendor MCP servers

No vendor is shipping an MCP server that knows your infrastructure.
Which CloudWatch log group corresponds to `payments-prod`? That depends on your
naming conventions. Which CodePipeline pipeline deploys to it? That depends on
your setup. Who owns it? That depends on your org structure.

Essentially, a custom MCP server can be designed to encode infrastructure knowledge your
team already has, in a place your agents can actually reach.
The business case is straightforward. Mean time to resolution (MTTR) is a
metric every engineering leader tracks, and tool fragmentation is one of its
biggest hidden costs. The time an engineer spends assembling context from five
sources before they can start resolving is real work. A custom MCP server doesn't change the underlying
systems. It eliminates the assembly tax.

**Note:** Obviously this is just one potential case for a custom MCP server, the pattern is general, so get as creative as wanted. You could build a custom MCP server for cost anomalies, security findings, deployment previews, capacity planning, or anything else that requires reasoning across multiple sources of truth.

## How a custom incident-context MCP server fits into your agent stack

With three vendor MCP servers configured alongside a custom one, an on-call
engineer's conversation could look like this:

```
Engineer: "PagerDuty just paged me for payments-prod. What's going on?"

Agent calls:
  → PagerDuty MCP   get_incident()       incident details, severity, alert source
  → Custom MCP      correlate_incident_context("payments-prod")
                      ↳ CloudWatch Logs Insights query (last 2 hours, errors only)
                      ↳ CodePipeline execution history (last 2 hours)
                      ↳ service owner from registry
  → Atlassian MCP   get_page()           Confluence runbook for payments-prod

Agent returns:
  Incident summary + correlated log errors + recent deployments + runbook first steps
```

The agent makes three MCP calls, reasons across the combined result, and
surfaces a structured incident summary, without the engineer stitching a single answer together by hand.

In this case the custom MCP exposes one primary tool: `correlate_incident_context`. Internally,
it does three things: resolves the service name against a registry to get log
group names and pipeline identifiers, runs a CloudWatch Logs Insights query
scoped to the incident window, and fetches recent CodePipeline executions for that
service. Three API calls, composed into one agent-callable function, returning
structured data the agent can reason about in context.

The reason this is one tool rather than three is deliberate. CloudWatch Logs
Insights is asynchronous: you start a query, poll until it completes, then
fetch the results. That polling loop is not something you want an agent
orchestrating across multiple separate tool calls. The MCP server handles the
async complexity; the agent sees a clean, synchronous result.

## Building a custom MCP server in Python: a basic example

The [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
provides a `FastMCP` class that handles the protocol.
A tool is a decorated function: its docstring is what the model reads to decide
when and how to call it, so write it as you would documentation for a capable
colleague who doesn't know your infrastructure.

This example uses AWS, but the same pattern applies to any stack: swap the CloudWatch and CodePipeline calls for your own observability and CI/CD providers.

**Prerequisites:**
- Python 3.10+
- `pip install mcp boto3`
- AWS credentials configured: `~/.aws/credentials`, environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION`), or an IAM role if running on AWS
- AWS resources tagged with a `Service` key (e.g. your log groups and pipelines tagged `Service=payments-prod`)

```python
import boto3
import time
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("incident-context")

logs_client = boto3.client("logs")
pipeline_client = boto3.client("codepipeline")

# In production, derive from resource tags via the Resource Groups Tagging API.
SERVICE_REGISTRY = {
    "payments-prod": {
        "log_group": "/ecs/payments/production",
        "pipeline": "payments-service-deploy",
        "owner": "platform-team",
    }
}


@mcp.tool()
def correlate_incident_context(
    service_name: str,
    incident_window_minutes: int = 120,
) -> dict:
    """
    Given a PagerDuty service name, returns correlated infrastructure context:
    recent log errors and recent deployments scoped to the incident window.
    Use this when an alert fires to understand what changed and what the logs show.
    Do not use this for services not in the registry: it will return an error.
    """
    registry_entry = SERVICE_REGISTRY.get(service_name)
    if not registry_entry:
        return {"error": f"Service '{service_name}' not found in registry"}

    end_ms = int(time.time() * 1000)
    start_ms = end_ms - (incident_window_minutes * 60 * 1000)

    # CloudWatch Logs Insights is async: start query, poll, fetch results
    query_id = logs_client.start_query(
        logGroupName=registry_entry["log_group"],
        startTime=start_ms // 1000,
        endTime=end_ms // 1000,
        queryString=(
            "fields @timestamp, @message"
            " | filter @message like /ERROR/"
            " | sort @timestamp desc"
            " | limit 20"
        ),
    )["queryId"]

    for _ in range(60):
        result = logs_client.get_query_results(queryId=query_id)
        if result["status"] in ("Complete", "Failed", "Cancelled"):
            break
        time.sleep(1)
    else:
        return {"error": "CloudWatch Logs Insights query timed out after 60s"}

    if result["status"] != "Complete":
        return {"error": f"Log query {result['status'].lower()}"}

    log_errors = [
        {col["field"]: col["value"] for col in row}
        for row in result.get("results", [])
    ]

    executions = pipeline_client.list_pipeline_executions(
        pipelineName=registry_entry["pipeline"], maxResults=10
    )["pipelineExecutionSummaries"]

    recent_deploys = [
        {
            "status": e["status"],
            "started": e["startTime"].isoformat(),
            "trigger": e.get("trigger", {}).get("triggerType", "unknown"),
        }
        for e in executions
        if e["startTime"].timestamp() * 1000 >= start_ms
    ]

    return {
        "service": service_name,
        "owner": registry_entry["owner"],
        "window_minutes": incident_window_minutes,
        "recent_errors": log_errors,
        "recent_deployments": recent_deploys,
    }


if __name__ == "__main__":
    mcp.run()
```

A few things worth noting. The `SERVICE_REGISTRY` dict is a placeholder. In
production, replace it with a query to the
[Resource Groups Tagging API](https://docs.aws.amazon.com/resourcegroupstagging/latest/APIReference/overview.html):
query by `Service=payments-prod` and resolve the log group and pipeline from the
returned ARNs. The mapping stays current automatically as your infrastructure changes.

The `incident_window_minutes` parameter defaults to 120 but the agent can narrow
it. If it calls the PagerDuty MCP first and retrieves the incident's `created_at`
timestamp, it can calculate a precise window rather than a fixed two-hour lookback.
That scopes the log query and deployment lookup to the actual incident window,
which reduces noise significantly.

The async CloudWatch Logs Insights pattern (start, poll, fetch) is the reason
this is worth wrapping into a tool. A two-step API with a polling loop is exactly
the kind of orchestration you don't want to push into the agent itself. The MCP
server handles it; the agent receives a clean, structured result it can reason
about immediately.

The tool's docstring does real work. The model reads it to decide when to call
this tool and what arguments to pass. "Use this when an alert fires to understand
what changed and what the logs show" and "Do not use this for services not in the
registry" are not documentation padding. They are the instruction set that makes
the tool's behaviour predictable.

Running the server locally is one command: `python server.py`. To add it to Claude
Code, add this entry to `.claude/settings.json` alongside your other MCP servers:

```json
{
  "mcpServers": {
    "incident-context": {
      "command": "python",
      "args": ["/path/to/server.py"]
    }
  }
}
```

The `command`/`args` config runs the server as a local child process, so each engineer needs the file on their machine. The simplest team approach is to keep `server.py` in a shared internal repo that everyone clones and runs locally. If you want a single centralised deployment instead, AWS publishes [prescriptive guidance on MCP hosting strategies](https://docs.aws.amazon.com/prescriptive-guidance/latest/mcp-strategies/mcp-hosting-strategy.html) covering Lambda, ECS Fargate, and Bedrock AgentCore as options.

## What You Get

Less manual stitching at 2am and an MTTR that falls without touching the underlying
system. This frees up senior engineers for the work they were actually hired to do.

Of course, you could go one step further and leave it to your agents ...
