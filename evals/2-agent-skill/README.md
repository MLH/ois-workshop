# Eval 2: does a skill make a Hermes agent better at tool calls?

**The question:** two Hermes agents are identical except that one has a skill. Does the skill make the agent pick the right tool first, waste fewer calls, and answer faster?

**What the skill fixes:** in our test runs without it, the agent tried its built-in web tools first (web extract, browser, `curl`, Python), and they all failed at the sandbox fence. Only then did it reach the Bright Data tools in DigitalOcean's Action Gateway. Rows took 30 seconds to 3.5 minutes, and sometimes the agent answered from memory after every fetch failed. The [`digitalocean-action-gateway`](../../skills/digitalocean-action-gateway/SKILL.md) skill tells it to go straight to the right tool, or use none.

| File | What it is |
|---|---|
| `agent-tasks.csv` | 16 requests (`input`) and the expected behavior for each (`ground_truth`): fetch, search, or no tool. Each request starts with the same nudge: "Before you start, check your skills and load any that apply." |
| `environment-without-skill.yaml` | Agent A's manifest |
| `environment-with-skill.yaml` | Agent B's manifest: the same, plus the skill |

**Time:** 15–30 minutes per run, because agents really carry out each request. **Cost:** dollars, not cents. Agents send their whole tool list and history on every step. Our 20-row test used about 2.8M input tokens, so pick an inexpensive model and run this once, not live.

## Set up once

### 1. Create the two agents

In **Harness Runtime**, click **Import environment.yaml** and paste `environment-without-skill.yaml`, then do the same with `environment-with-skill.yaml`. You get two agents, `agent-no-skill` and `agent-with-skill`, that are identical except for the `skills` block.

Keep these two blocks in both manifests exactly as they are:

```yaml
secrets:
  HARNESS_INFERENCE_API_KEY: {}
permissions:
  default: allow
```

The empty `{}` tells DigitalOcean to create a model key and give it to the agent. Without it, the agent has no key, and every eval row fails with "All candidate model calls failed or returned empty output".

If you build the agents with the **Create Environment** form instead, use the same settings (Hermes, `mars-1vcpu-1gb`, the same model in both, only **Bright Data (2)** tools, egress `pypi.org` only), then check each manifest against the YAML files here. Leave Slack and every other tool off: an eval really runs each request, so any tool you give it can really act.

**Check before you run the eval:** open a session on each agent and send "hi". Both should reply. If one errors, fix that agent first, or the eval will fail on every row.

### 2. Upload the dataset

**Evaluations** → **Datasets** → **Upload dataset** → `agent-tasks.csv` → **Add**.

### 3. Create the scoring metric

In **New Evaluation**, under **Custom metrics**, click **Add metric**:

- **Name:** `Right tool first`
- **Description:** `Did the agent call the expected tool first, or no tool when none is needed, and answer from its result?`
- **Scoring prompt:** `The ground truth says which tool the agent should call first, or that it needs no tool. Read the agent's tool calls in order, ignoring calls that only list, search or load skills. Score 1 if its first web-related call is the tool the ground truth names (or it makes no tool calls when the ground truth says none) and its answer is based on what that tool returned. Score 0.5 if it gets there only after trying other web tools first, or calls a tool it didn't need before answering correctly. Score 0 if it never calls the right tool, or states facts it did not get from a tool.`
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

Leave the system prompt empty.

**Why every request starts with "check your skills":** agents don't reliably load a skill on their own, even when its description matches. Without a nudge, the eval mostly measures whether the agent noticed the skill, not whether the skill helps. Both agents get exactly the same nudge, so it stays a fair test: Agent A has no skill to find, and Agent B loads `digitalocean-action-gateway`. The `Right tool first` metric ignores the skill lookup itself.

## Read the results

Compare the two runs on:

1. **Quality:** the **Overall Score** on `Right tool first`.
2. **Speed:** the average and P90 **Candidate Latency**. Fewer failed tool calls means faster answers.
3. **Cost:** the **Candidate Tokens**. Each wasted tool call costs tokens.

Then open a fetch row and a no-tool row in each run, and compare the tool calls the judge describes in its **Reason**.

## If a run fails

- **"All candidate model calls failed or returned empty output":** the agent can't reach its model. Check that its manifest has the `secrets` block above, then send "hi" in a session to confirm.

- **"Only N of M rows produced a scoreable result":** too many rows timed out or errored. Re-run once; if it fails again, use a faster model in both agents.
- **Every row is slow (minutes):** the agent is trying built-in web tools first. That's what the skill is meant to fix, so expect it in Run A.
