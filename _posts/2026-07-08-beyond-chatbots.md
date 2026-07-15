---
title: "Beyond Chatbots: Making Your Product Usable by AI"
date: 2026-07-08 00:00:00 +0200
image: /assets/img/posts/beyond-chatbots/splash.png
categories: [AI & ML, Product Strategy]
tags: [mcp, ai-agents, product-strategy, ai-adoption, agentic-ai]
description: |
    Companies are trying to figure out what building "something with AI" should mean for their products. Many default to chatbots and copilots, but a more useful question is whether AI agents may become users of the product itself. If the answer is yes, MCP is one practical way to make existing product capabilities available to those agents.
---

*Co-written with Robert von Massow.*

## Everyone is building AI features

Right now many companies are in the 'chaotic' phase of their AI adoption. Internally, engineers started using whatever was close at hand: maybe one tried OpenAI models, the next used Claude, another team experimented with agent frameworks. Meanwhile, in more mature organisations, this has turned into something more structured: internal AI platforms, agent harnesses, governance models, or even environments where citizen developers can build their own skills.

Interestingly, when AI moves from internal productivity to external-facing products, that broad exploration narrows to a single question: where do we put the AI assistant? The answer is usually a chatbot on the website, a copilot inside the application, or an "Ask our AI" button next to existing documentation. That's not necessarily wrong, but it's a limited view of what AI can do for a product.

When talking to engineering teams about AI adoption, a simple question comes up often: "How large is your team?" The answer is typically something like: "Five developers and a product owner." The follow-up is: "And how many agents?"

That question tends to change the room. For some people it's surprising, for others, slightly uncomfortable, and for some it feels like a small revelation. They have used agents as tools, but they haven't thought about agents as participants that need to be planned for, managed, and designed around. Product teams may need a similar shift. The question isn't only: "What AI feature should we add to our product?" It's also: "Who, or what, will use our product next?"

## Your next user might be an AI agent

One possible answer is: an AI agent. An AI agent as a user doesn't mean the same thing for every product. For a documentation service, agent access may be purely informational: the agent reads the documentation, understands available options, and uses that knowledge to help a human achieve a goal.

For a project management tool, the interaction could be much more active. A developer may want an agent to pick up a ticket, work on the implementation, and move the ticket forward when the work is done. A product owner may want help writing new tickets or refining existing ones. A scrum master may want to analyse team metrics such as parallel work in progress, lead time, or bottlenecks.

For a travel platform, the agent interaction could be broader still. A human might describe the kind of holiday they want, let an agent research destinations, compare flights and accommodation, and eventually prepare a booking.

In contrast, an online bank is a useful counterexample. Just because something can be exposed to an agent doesn't mean it should be exposed without careful constraints. There are domains where the right answer may be read-only access, explicit human approval, or no agent access at all.

This isn't a completely new problem. It's a variation of the same question we already ask about human users: who should be allowed to do what, under which conditions, and with which audit trail? Different people will draw that line at different points. Some will be dangerously careless and hand over sensitive data without thinking. Others will be so cautious that they refuse to let an agent read a file on their machine. Product teams will have to design for that spectrum.

As agentic interactions become ever more relevant, a shift in thinking could be valuable: don't assume that every user of your product will be a human clicking through your UI. Product thinking has always involved understanding users more deeply than their immediate requests. If we only optimise for what existing users already ask for, we risk missing the next interaction model entirely. In an agentic world, the question is not only how to sell a product to humans, but also how that product becomes useful to the agents acting on their behalf.

## MCP turns your product into a capability

Once you've accepted that AI agents might be users of your product, the next question is practical: how do they actually interact with it?

This is where the Model Context Protocol (MCP) comes in. At its core, MCP is a standard that allows AI agents to discover and use the capabilities of external systems. Those capabilities may be as simple as retrieving information, or as powerful as creating records, executing workflows, invoking APIs, or even running code. Which capabilities are exposed, and under which permission set, is entirely up to the product.

Some companies have already recognised this shift. Rather than only making their products more intelligent, they are making them accessible to intelligent systems. GitHub's MCP server is a good example. Instead of limiting AI integration to GitHub Copilot, GitHub exposes its platform so that a wide range of AI agents can interact with repositories, issues, pull requests, and workflows through a standard interface.

That approach has another advantage. The agent ecosystem is evolving at remarkable speed. Today it might be Claude, ChatGPT, Cursor or another development environment. Tomorrow it may be an operating system, a workflow platform, or a product that doesn't yet exist. Supporting an open standard allows your product to participate in that ecosystem without betting on a single vendor or assistant.

Thinking about MCP in this way changes the discussion entirely. It's no longer a choice between building an AI feature or an MCP server. An MCP server can itself be an AI feature, one that serves a different kind of user. Whether or not MCP is the right answer for your product depends on a much more fundamental question: who are you building for?

## The question to ask before you build anything with AI

What we really want to drive home here is that it might be prudent to add one more question to your product strategy meetings:

 **Which parts of our product would users rather delegate to an AI agent than do themselves?**

If the answer is "none", that's a perfectly valid outcome. Not every product should expose itself to agents, and not every capability belongs in an agentic workflow. But those decisions should be deliberate rather than accidental.

If the answer is "some", then you're no longer discussing whether your product needs AI. You're discussing how your product participates in an ecosystem where AI agents are becoming another class of user. Today, MCP is the leading open standard for making that possible. The next time someone asks for "something with AI", don't start by discussing models. Start by asking who your next users might be. In the agentic era, some of them may never click a button.

---

### Read more about MCP and agentic interactions

**[Building Your Own MCP Server](/posts/building-your-own-mcp-server/)**: when vendor MCP servers aren't enough, and how to build a custom one that fills the gap.
