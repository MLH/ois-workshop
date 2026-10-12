# Evals

An eval runs the same set of requests through a candidate (a model, a router or an agent), and a judge model scores every answer against the right one. It replaces "it seemed better" with a number you can compare.

This folder has two:

| Eval | Compares | Candidate | Time | Cost |
|---|---|---|---|---|
| [1 · Compare two models](1-open-vs-closed/) | Two models (open, or closed if your plan allows) on support-ticket triage | Models | Minutes | Cents |
| [2 · Agent with vs. without a skill](2-agent-skill/) | Two Hermes agents, identical except for one skill, on tool calls | Agents | 15–30 min per run | Dollars |

Start with eval 1. It's fast, and it shows the whole Evaluations flow: dataset, candidate, judge, metric, results. Eval 2 uses the same flow on real agents.

## How to build your own

1. **Pick one question,** such as "Is the cheap model good enough?" or "Does this skill help?"
2. **Write 15–50 rows** in a CSV with two columns: `input` (the request) and `ground_truth` (the right answer, or the right behavior).
3. **Hold everything else equal** between the runs you compare: same dataset, judge, metric and prompt. Change only the thing you're testing.
4. **Write a custom metric** that scores exactly what you care about. The built-in metrics are general-purpose and can score short or structured answers oddly.
5. **Read the judge's reasons** on the rows that failed. That's where you learn what to change.
