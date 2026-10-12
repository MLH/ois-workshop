# Eval 2: does a skill make a Hermes agent better at tool calls?

**The question:** two Hermes agents are identical except that one has the `digitalocean-action-gateway` skill. Does the skill make the agent pick the right tool first, waste fewer calls, and answer faster?

**What the skill fixes:** in our test runs without it, the agent tried its built-in web tools first (web extract, browser, `curl`, Python), and they all failed at the sandbox fence. Only then did it reach the Bright Data tools in DigitalOcean's Action Gateway. Rows took 30 seconds to 3.5 minutes, and sometimes the agent answered from memory after every fetch failed. The [`digitalocean-action-gateway`](../../skills/digitalocean-action-gateway/SKILL.md) skill tells it to go straight to the right tool, or use none.

**How the agent finds the skill:** both agents also have a small [`start-here`](../../skills/start-here/SKILL.md) skill, and every request tells the agent to load it first. `start-here` lists the skills written for this environment, says when to load each one, and says they win over built-in skills. It's the one skill you always name; it routes to the rest.

| File | What it is |
|---|---|
| `agent-tasks.csv` | 16 requests (`input`) and the expected behavior for each (`ground_truth`): fetch, search, or no tool. Each request starts with the same nudge: "First, load your start-here skill." |
| `environment-without-skill.yaml` | Agent A's manifest, with only `start-here` |
| `environment-with-skill.yaml` | Agent B's manifest: the same, plus `digitalocean-action-gateway` |

**Time:** 15–30 minutes per run, because agents really carry out each request. **Cost:** dollars, not cents. Agents send their whole tool list and history on every step. Our 20-row test used about 2.8M input tokens, so pick an inexpensive model and run this once, not live.

## Set up once

### 1. Create the two agents

First, create a model key the agents will use. Go to **Model Access Keys** (under Inference) → **Create model access key**, leave **All models** on, pick **No VPC network**, and copy the `doo_v1_…` value. It's shown only once.

Then, in **Harness Runtime**, click **Import environment.yaml** and paste `environment-without-skill.yaml`. Replace `<your model access key>` with your key, and import. Do the same with `environment-with-skill.yaml`. You get two agents, `agent-no-skill` and `agent-with-skill`. Both have `start-here`; only `agent-with-skill` has `digitalocean-action-gateway`.

Import needs a real key in `secrets`. With an empty value it refuses ("Secret … is declared but has no value"), and with no `secrets` block at all the agent has no key, so every eval row fails with "All candidate model calls failed or returned empty output". Keep the key out of screenshots, streams and git.

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

**Why every request says "load your start-here skill":** agents don't reliably load a skill on their own, even when its description matches. A vague nudge isn't enough either: told to "check your skills and load any that apply", Hermes picked its built-in `blocked-page-recovery` skill and tried `curl`, a browser and the Wayback Machine before giving up. Naming one fixed skill in every request is reliable, and `start-here` then points the agent at the right skill for the task. Both agents get the same nudge and the same `start-here`, so the only difference is `digitalocean-action-gateway`: Agent A finds it isn't installed and carries on, and Agent B loads it. The `Right tool first` metric ignores skill lookups.

## Read the results

Compare the two runs on:

1. **Quality:** the **Overall Score** on `Right tool first`.
2. **Speed:** the average and P90 **Candidate Latency**. Fewer failed tool calls means faster answers.
3. **Cost:** the **Candidate Tokens**. Each wasted tool call costs tokens.

Then open a fetch row and a no-tool row in each run, and compare the tool calls the judge describes in its **Reason**.

## If a run fails

- **"All candidate model calls failed or returned empty output":** the agent can't reach its model. Check that it was imported with a real key in `secrets`, then send "hi" in a session to confirm.

- **"Only N of M rows produced a scoreable result":** too many rows timed out or errored. Re-run once; if it fails again, use a faster model in both agents.
- **Every row is slow (minutes):** the agent is trying built-in web tools first. That's what the skill is meant to fix, so expect it in Run A.
