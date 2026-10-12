# Eval 1: open model vs. closed model

**The question:** on a real everyday task, how does a cheap open-weight model compare with a frontier closed model, on quality, cost and speed?

**The task:** support-ticket triage. Each row is one customer message. The model must pick one of six queues and an urgency, then reply with JSON like `{"queue": "billing", "urgency": "normal"}`. There are 15 messages, each with the right answer.

| File | What it is |
|---|---|
| `ticket-triage.csv` | 15 customer messages (`input`) and the right queue and urgency for each (`ground_truth`) |
| `system-prompt.txt` | The triage instructions both models get: the queues, the urgency levels and the rules |

**Time:** about 5 minutes of setup, then a minute or two per run. **Cost:** cents.

## Set up once

### 1. Upload the dataset

In the DigitalOcean console, go to **Evaluations** (under Inference) → **Datasets** → **Upload dataset**, choose `ticket-triage.csv`, and click **Add**.

### 2. Make the closed model available to Evaluations

Evaluations lists open models under **Serverless Inference**. Closed models such as Anthropic Claude and OpenAI GPT appear in the Model Catalog, but not in that list. To evaluate one, put it behind a router that only ever uses that model:

1. Go to **Inference Router** → **Create Router**. Name it `closed-claude-sonnet`.
2. Add no tasks. Click **Add Fallback Models**, select **Anthropic Claude Sonnet 5.5** (or another closed model), click **Save**, then **Create Router**.

With no tasks, every request goes to the fallback, so the router is that one model. If the form insists on a task, add one custom task named `everything`, described as "Every request", with only that model.

### 3. Create the scoring metric

In **New Evaluation**, under **Custom metrics**, click **Add metric**:

- **Name:** `Exact match`
- **Description:** `Did the model pick the same queue and urgency as the ground truth?`
- **Scoring prompt:** `The ground truth is a JSON object with a queue and an urgency. Score 1 if the output names the same queue and the same urgency, ignoring formatting, key order and extra text. Otherwise score 0.`
- Tick **Requires ground truth**, then click **Create**.

You only create the metric once. Later runs pick it from the list.

## Run it

Start two evaluations with the **same dataset, judge, metric and system prompt**. Only the candidate changes.

| | Run A: open | Run B: closed |
|---|---|---|
| **Candidate type** | Serverless Inference | Model Router |
| **Candidate** | MiniMax M2.5 (or DeepSeek V4 Flash) | `closed-claude-sonnet` |
| **System prompt** | all of `system-prompt.txt` | all of `system-prompt.txt` |
| **Dataset** | `ticket-triage.csv` | `ticket-triage.csv` |
| **Judge** | Deepseek V4 Pro | Deepseek V4 Pro |
| **Star metric** | `Exact match` | `Exact match` |
| **Run name** | `open-minimax` | `closed-sonnet` |

## Read the results

Open both runs side by side (or use **Compare Evaluation**) and look at three things:

1. **Quality:** the **Overall Score**. How many of the 15 did each get right?
2. **Cost:** the **Candidate Tokens**, multiplied by each model's price in the Model Catalog. For example, MiniMax M2.5 is $0.30 / $1.20 per million input / output tokens, and Claude Sonnet 5.5 is $2.00 / $10.00.
3. **Speed:** the **Candidate Latency**.

Then open the rows either model got wrong and read the judge's **Reason**. The interesting rows are usually the judgment calls: a hacked account (urgent), or a refund caused by a billing error (refunds, not billing).

If the open model is close on quality at a fraction of the cost, it's a good default, and a router can send only the hard cases to the closed model. If it isn't close, you've learned where the closed model earns its price.
