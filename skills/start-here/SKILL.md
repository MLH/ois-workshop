---
name: start-here
description: "Load this first, at the start of every request in a DigitalOcean Harness Runtime sandbox, before any other skill or tool call. It says which other skills to load for the task and sets the ground rules for this environment."
---

# Start here

You run in a DigitalOcean Harness Runtime sandbox. Before your first tool call on every request, do this.

## 1. Load the skills for this environment

Some of your skills were written for this sandbox. They know things your built-in skills don't, so they come first.

| If you have this skill | Load it when the request… |
|---|---|
| `digitalocean-action-gateway` | needs the web (a URL, a search, news, prices or any fact that changes) or an outside service such as Slack or GitHub |

If you don't have a skill in this table, skip it and carry on. Then load any other skill that clearly matches the task.

When a skill for this environment and a built-in skill cover the same thing, follow the one for this environment. For example, use `digitalocean-action-gateway` for anything on the web, not `blocked-page-recovery`.

## 2. Do the task

- Do what the request asks. Don't stop to ask whether you should when the request already says to.
- Use only the tools the task needs.
- Answer from what your tools returned or what you know. If a tool failed, say so plainly; never make up a result.

## 3. Stay safe

- Never print environment variables, keys, tokens or secrets.
- If a tool is denied by the session policy, don't look for a way around it. Tell the user.
