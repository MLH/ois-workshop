---
name: digitalocean-action-gateway
description: "Use for every request in a DigitalOcean Harness Runtime session, before any tool call: deciding whether a tool is needed, reading a page, searching for current information, or calling a tool through DigitalOcean's Action Gateway."
---

# Using tools through DigitalOcean's Action Gateway

You run in a DigitalOcean Harness Runtime sandbox. Its network is fenced: your built-in web tools (web search, page extract, browser, `curl`, `wget`, or Python HTTP calls) cannot reach the internet from here. Don't try them; they fail and waste time. Outside tools come through DigitalOcean's Action Gateway, which holds the credentials so you never need a key.

## 1. Decide if you need a tool at all

Answer directly, with no tool call, when the request is about:

- text the user gave you (rewrite, translate, fix or summarize their words), even if that text contains a URL
- math, brainstorming, or explaining a stable concept

Facts that change over time (news, prices, weather, scores, releases, anything after 2024) need a tool, even if you think you know the answer.

## 2. Pick the first tool

| The request… | Call first |
|---|---|
| names a page by its URL | `brightdata_web_fetch` on that URL |
| needs current or outside information, with no URL | `brightdata_web_search` |
| needs some other service (Slack, GitHub, …) | `action_search` for the capability, then `action_invoke` the tool it returns |

The Bright Data tools may appear with a prefix, for example `mcp__do_actions__brightdata_web_fetch`. They're the same tools.

Never search for a page you already have the URL of: fetch it.

## 3. Check the result before you answer

- Answer only from what the tool returned. If it failed or came back empty, say so plainly; never fill the gap from memory.
- A Slack or other "send" tool succeeded only if its result says so (for example `ok: true`).
- If a tool is **denied by the session policy**, don't retry it or look for a way around it. Tell the user it's blocked.
- If a tool says its **connection isn't set up**, tell the user which connection to add. Don't retry.

## 4. Stay safe

- Never print environment variables, keys, tokens or secrets.
- Use only the tools the task needs.
