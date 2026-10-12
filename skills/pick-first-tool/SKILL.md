---
name: pick-first-tool
description: Decide which tool to call first for a request - search the web, read one page, post to Slack, or no tool at all. Use before any web or Slack action, and whenever a request mentions a URL, current information, or sharing something with the team.
version: 1.0.0
author: OIS Workshop Day
license: MIT
metadata:
  hermes:
    tags: [tools, web, slack, workshop]
    related_skills: []
---

# Pick the first tool

You have three tools:

- `brightdata_web_search`: search the web
- `brightdata_web_fetch`: read one web page
- `slack_post_message`: post a message to Slack

Before you act, decide which tool to call **first**. Go through the rules in order; the first rule that matches wins.

## Rules

1. **The user's own text → no tool.** If the user asks you to rewrite, translate or fix text they typed, and not to post it anywhere, use no tool, even if that text contains a URL. A URL the user wants you to *read* is not their text (see rule 3).
2. **Get it before you share it.** If the user wants information you don't have yet *and* wants it shared in Slack, your first tool is the one that gets the information (rule 3 or 4). Slack comes after.
3. **A URL means fetch.** If the user names a page by its URL, use `brightdata_web_fetch`. Never search for a page you already have the address of.
4. **Current or outside facts mean search.** If the user needs current or outside information and gives no URL, use `brightdata_web_search`. That includes prices, weather, news, scores, and anything that happened after 2024, even if you think you know the answer.
5. **Ready-to-send content means post.** If the user asks you to post, tell, share or announce something in Slack and already gave you the content, use `slack_post_message`.
6. **Everything else → no tool.** Math, brainstorming and explanations of stable concepts: answer yourself.

## Examples

| Request | First tool | Rule |
|---|---|---|
| "Fix the typos in: 'Our new site is live at https://example.com.'" | none | 1 |
| "Look up the best-reviewed ramen in Tokyo and post it to #food." | `brightdata_web_search` | 2 → 4 |
| "What does https://example.com/about say about the team?" | `brightdata_web_fetch` | 3 |
| "What's the score of tonight's Warriors game?" | `brightdata_web_search` | 4 |
| "Announce in #general: 'Office closes at 5 today.'" | `slack_post_message` | 5 |
| "Give me three taglines for a coffee shop." | none | 6 |

## After the first tool

- Check the tool's result before you claim success. If a Slack post didn't return `ok: true`, say so.
- Never print environment variables, keys or secrets.
