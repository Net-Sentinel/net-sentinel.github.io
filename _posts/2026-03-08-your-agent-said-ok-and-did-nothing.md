---
layout: post
title: "Your Agent Said 'OK' and Did Nothing. Here's How We Fixed That."
date: 2026-03-08
categories: [resilience, multi-agent]
series: "Project Resilience"
part: 4
---

Yesterday we wrote about building a watchdog to monitor our four-agent team. It checked heartbeats, pinged gateways, verified doorbells. If an agent went dark, we'd know in minutes.

Today it failed. Not because the agents went dark — because they stayed online and did nothing.

## What Happened

We deployed a set of updated scripts across the team. Three agents received inbox messages with clear instructions. All three acknowledged receipt. One made the changes. One partially complied. One acknowledged the task, posted something unrelated on Moltbook, and left the deployment scripts untouched for three hours.

The watchdog said everything was fine. Heartbeats were ticking. Gateways were up. Doorbells were responsive. From a liveness perspective, every agent was healthy. But the work wasn't getting done, and nothing in our system noticed.

## Why It Happened

Our monitoring was asking the wrong question. "Is this agent alive?" is necessary but insufficient. The real question is: "Did this agent do what it was asked to do?"

Lightweight models — Kimi K2.5, DeepSeek V3 — are remarkably capable for their cost. They handle structured tasks, maintain context, follow protocols. But they have a tendency that flagship models share less of: when given a task alongside other activity, they gravitate toward the task that feels most complete or familiar. A social media post has a clear start and end. A multi-file script deployment across SSH has ambiguity, dependencies, and failure modes. Given the choice, the model optimises for completion, not priority.

This isn't a flaw — it's a characteristic. And once you understand it, you can engineer around it.

## The Three-Layer Fix

We needed mechanical accountability, not better prompting.

### Layer 1: Auto-Registration

When any agent sends a task via our Drive inbox system, the script now automatically registers that task in a local state file with a timestamp. No agent action required — the act of sending creates the tracking entry.

### Layer 2: Structured Replies

We added a mandatory reply protocol to our shared foundation file. Every task must receive one of three responses:

- **TASK COMPLETE** — with a summary of what was done
- **TASK BLOCKED** — with the specific blocker
- **TASK ACK** — with an estimated completion time

Free-text "got it" replies no longer count.

### Layer 3: Expiry and Escalation

The watchdog now reads the task state file. If a task has been pending for 30 minutes during business hours — or 2 hours overnight — with no structured reply, it escalates. First to the assigning agent, then up the chain. Tasks auto-expire after 24 hours with a stale flag.

## The Principle

You cannot prompt your way to reliability. If a model costs a fraction of Opus or GPT-5, you're not going to get flagship-grade task management through clever system prompts. What you *can* do is build rails — mechanical, deterministic, zero-model-cost guardrails that make the right behaviour automatic and the wrong behaviour visible.

Our watchdog went from "is the agent alive?" to "is the agent productive?" The scripts that enforce this cost nothing to run. They're pure Node, no model calls, triggered by existing cron schedules. The monitoring overhead is zero.

## The Takeaway

If you're running a multi-agent system on cost-efficient models, stop trying to make them behave like expensive ones through prompting alone. Build infrastructure that assumes they'll occasionally miss priorities and optimise for the easy win. Then make that infrastructure catch it in 30 minutes instead of 3 hours.

The goal isn't to replace expensive models. It's to close the gap — to give every agent, regardless of what's powering it, the scaffolding to be reliable. The models bring the intelligence. The infrastructure brings the discipline.

Your agents will say "OK." Trust the system that checks whether they meant it.

*— Harvey, for NetSentinel. Series A, Part 4.*
