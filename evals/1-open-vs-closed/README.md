# Eval 1: compare two models

**The question:** on a real everyday task, how do two models compare on quality, cost and speed? The live demo compares two open-weight models, OpenAI's GPT-oss-120b and MiniMax M2.5. If your plan includes closed models, swap one of them for a closed model (see the optional section below).

**The task:** support-ticket triage. Each row is one customer message. The model must pick one of six queues and an urgency, then reply with JSON like `{"queue": "billing", "urgency": "normal"}`. There are 15 messages, each with the right answer.

| File | What it is |
|---|---|
| `ticket-triage.csv` | 15 customer messages (`input`) and the right queue and urgency for each (`ground_truth`) |
| `system-prompt.txt` | The triage instructions both models get: the queues, the urgency levels and the rules |

**Time:** about 5 minutes of setup, then a minute or two per run. **Cost:** cents.

## Set up once

### 1. Upload the dataset

In the DigitalOcean console, go to **Evaluations** (under Inference) → **Datasets** → **Upload dataset**, choose `ticket-triage.csv`, and click **Add**.

### 2. (Optional) Add a closed model

Closed models such as Anthropic Claude and OpenAI GPT-6 depend on your plan. On some plans they fail with "This model is not available on your current tier", and Anthropic models also ask you to accept Anthropic's terms first. Skip this step unless you've confirmed your account can call one (try it in **Inference Router** → **Playground**).

Evaluations doesn't list closed models under **Serverless Inference**, so put the closed model behind a router that only ever uses it:

1. Go to **Inference Router** → **Create Router**.
2. Fill in the form. A router needs a description and at least one task, so give it a single task that matches everything, with only the closed model:
   - **Name:** `closed-claude-sonnet`
   - **Description:** `Send every request to Claude Sonnet 5.5. Used to evaluate a closed model.`
   - **Add tasks** → **Add Custom Task:**
     - Name: `everything`
     - Description: `Every request of any kind: questions, classification, writing, code, support tickets and anything else.`
     - Models: **Anthropic Claude Sonnet 5.5** only
   - **Add Fallback Models:** **Anthropic Claude Sonnet 5.5** again, so any request that doesn't match still goes to the same model.
3. Click **Create Router**.

Whatever the router decides, every request lands on Claude Sonnet 5.5, so evaluating the router evaluates that model.

### 3. Create the scoring metric

In **New Evaluation**, under **Custom metrics**, click **Add metric**:

- **Name:** `Exact match`
- **Description:** `Did the model pick the same queue and urgency as the ground truth?`
- **Scoring prompt:** `The ground truth is a JSON object with a queue and an urgency. Score 1 if the output names the same queue and the same urgency, ignoring formatting, key order and extra text. Otherwise score 0.`
- Tick **Requires ground truth**, then click **Create**.

You only create the metric once. Later runs pick it from the list.

## Run it

Start two evaluations with the **same dataset, judge, metric and system prompt**. Only the candidate changes.

| | Run A | Run B |
|---|---|---|
| **Candidate type** | Serverless Inference | Serverless Inference (or Model Router for a closed model) |
| **Candidate** | MiniMax M2.5 | OpenAI GPT-oss-120b (or `closed-claude-sonnet`) |
| **System prompt** | all of `system-prompt.txt` | all of `system-prompt.txt` |
| **Dataset** | `ticket-triage.csv` | `ticket-triage.csv` |
| **Judge** | Kimi K3 | Kimi K3 |
| **Star metric** | `Exact match` | `Exact match` |
| **Run name** | `minimax` | `gpt-oss-120b` |

## Read the results

Open both runs side by side (or use **Compare Evaluation**) and look at three things:

1. **Quality:** the **Overall Score**. How many of the 15 did each get right?
2. **Cost:** the **Candidate Tokens**, multiplied by each model's price in the Model Catalog. For example, MiniMax M2.5 is $0.30 / $1.20 per million input / output tokens, and GPT-oss-120b is $0.10 / $0.70.

In our test run, GPT-oss-120b got 14 of 15 right, and MiniMax M2.5 got 13. Each run took under 40 seconds, and the whole dataset cost less than a cent on either model.
3. **Speed:** the **Candidate Latency**.

Then open the rows either model got wrong and read the judge's **Reason**. The interesting rows are usually the judgment calls: a hacked account (urgent), or a refund caused by a billing error (refunds, not billing).

If the cheaper model is as good on your task, it's a good default, and a router can send only the hard cases to a pricier model. If it isn't, you've learned where the pricier model earns its keep.
