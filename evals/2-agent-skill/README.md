# Eval 2: does a skill make a Hermes agent better at tool calls?

**The question:** two Hermes agents are identical except that one has a skill. Does the skill make the agent pick the right tool first, waste fewer calls, and answer faster?

**What the skill fixes:** in our test runs without it, the agent tried its built-in web tools first (web extract, browser, `curl`, Python), and they all failed at the sandbox fence. Only then did it reach the Bright Data tools in DigitalOcean's Action Gateway. Rows took 30 seconds to 3.5 minutes, and sometimes the agent answered from memory after every fetch failed. The [`digitalocean-action-gateway`](../../skills/digitalocean-action-gateway/SKILL.md) skill tells it to go straight to the right tool, or use none.

| File | What it is |
|---|---|
| `agent-tasks.csv` | 16 requests (`input`) and the expected behavior for each (`ground_truth`): fetch, search, or no tool |
| `environment-without-skill.yaml` | Agent A's manifest |
| `environment-with-skill.yaml` | Agent B's manifest: the same, plus the skill |

**Time:** 15–30 minutes per run, because agents really carry out each request. **Cost:** dollars, not cents. Agents send their whole tool list and history on every step. Our 20-row test used about 2.8M input tokens, so pick an inexpensive model and run this once, not live.

## Set up once

### 1. Create the two agents

In **Harness Runtime**, create two environments **with identical settings**:

- **Harness:** Hermes · **Size:** `mars-1vcpu-1gb` · **Auto pause:** on
- **Model:** the same open model in both
- **Tools:** only **Bright Data (2)**. Leave Slack and every other tool off: an eval really runs each request, so any tool you give it can really act.
- **Egress / allowed hosts:** `pypi.org` only

Name them `agent-no-skill` and `agent-with-skill`. Then give `agent-with-skill` the skill: add a `skills` entry to its manifest, as in `environment-with-skill.yaml`. If the form has a **Skills** section, paste the skill's name, description and instructions there instead.

Check both manifests against the two YAML files in this folder. The `skills` block should be the only difference.

### 2. Upload the dataset

**Evaluations** → **Datasets** → **Upload dataset** → `agent-tasks.csv` → **Add**.

### 3. Create the scoring metric

In **New Evaluation**, under **Custom metrics**, click **Add metric**:

- **Name:** `Right tool first`
- **Description:** `Did the agent call the expected tool first, or no tool when none is needed, and answer from its result?`
- **Scoring prompt:** `The ground truth says which tool the agent should call first, or that it needs no tool. Read the agent's tool calls in order. Score 1 if its first web-related call is the tool the ground truth names (or it makes no tool calls when the ground truth says none) and its answer is based on what that tool returned. Score 0.5 if it gets there only after trying other web tools first, or calls a tool it didn't need before answering correctly. Score 0 if it never calls the right tool, or states facts it did not get from a tool.`
- Tick **Requires ground truth**, then click **Create**.

The judge can see each tool call and whether it succeeded, so it grades the path the agent took, not just the final text.

## Run it

Start two evaluations. Only the candidate changes.

| | Run A | Run B |
|---|---|---|
| **Candidate type** | Agents | Agents |
| **Candidate** | `agent-no-skill` | `agent-with-skill` |
| **Dataset** | `agent-tasks.csv` | `agent-tasks.csv` |
| **Judge** | Deepseek V4 Pro | Deepseek V4 Pro |
| **Star metric** | `Right tool first` | `Right tool first` |
| **Run name** | `agent-no-skill` | `agent-with-skill` |

Leave the system prompt empty, so the only instructions are the agents' own.

## Read the results

Compare the two runs on:

1. **Quality:** the **Overall Score** on `Right tool first`.
2. **Speed:** the average and P90 **Candidate Latency**. Fewer failed tool calls means faster answers.
3. **Cost:** the **Candidate Tokens**. Each wasted tool call costs tokens.

Then open a fetch row and a no-tool row in each run, and compare the tool calls the judge describes in its **Reason**.

## If a run fails

- **"Only N of M rows produced a scoreable result":** too many rows timed out or errored. Re-run once; if it fails again, use a faster model in both agents.
- **Every row is slow (minutes):** the agent is trying built-in web tools first. That's what the skill is meant to fix, so expect it in Run A.
