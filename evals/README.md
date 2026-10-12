# Eval kit: does a skill make the agent pick the right tool?

A **skill** is a folder with a `SKILL.md` file: a name, a one-line description of when to use it, and instructions. Agents like Hermes load a skill when a request matches its description, so you can teach an agent your team's way of doing something once and reuse it.

This kit tests the skill in [`../skills/pick-first-tool/SKILL.md`](../skills/pick-first-tool/SKILL.md). The skill teaches the agent which tool to call first: search the web, read one page, post to Slack, or no tool at all.

You run the same 20 requests through the same model twice, once without the skill and once with it, and a judge model scores every answer.

| File | What it is |
|---|---|
| `tool-pick.csv` | 20 requests (`input`) and the right first tool for each (`expected_response`) |
| `agent-prompt.txt` | The agent's base instructions: the three tool names and one-line descriptions, nothing more |
| `../skills/pick-first-tool/SKILL.md` | The skill under test |

The test requests are different from the examples inside the skill, so the eval measures whether the rules carry over to new requests, not whether the model can copy an example.

## Run it (about 10 minutes)

1. In the DigitalOcean console, go to **Evaluations** (under Inference) and open the **Datasets** tab. Click **Upload dataset**, choose `tool-pick.csv`, and click **Add**.
2. Go back to **Runs** and click **New Evaluation**, then **Configure without a preset**.
3. **Candidate:** set the type to **Serverless Inference** and pick an open model.
4. **System prompt:** paste all of `agent-prompt.txt`.
5. **Dataset:** select `tool-pick.csv`.
6. **Judge and metrics:** pick a judge model. A different model from the candidate is best. Select **Ground Truth Faithfulness** and make it the **Star Metric**.
7. Under **Optional settings**, name the run `without-skill`, then click **Run Evaluation**.
8. Click **New Evaluation** again with the same model, dataset and judge. In **System prompt**, paste `agent-prompt.txt`, add a blank line, then paste the whole `SKILL.md`. Name the run `with-skill`.

   Why paste it? When an agent loads a skill, the skill's text goes into the model's context next to its instructions. Pasting it in does the same thing, so the eval tests the exact words your agent would see.

9. When both runs show **Completed**, open each one. Compare the **Overall Score**, and read the **Reason** column on the rows the first run got wrong.

## What to look for

Without the skill, the model knows the tool names but not your rules, so it tends to miss these:

- **"Rewrite this…" with a URL inside:** it fetches the URL. The skill says it's the user's own text, so no tool is needed.
- **"Find X and share it in Slack":** it goes straight to Slack. The skill says to get the information first.
- **"Who won the most recent Super Bowl?":** it answers from memory. The skill says to search for anything after 2024.

Your numbers will vary by model. The point is to measure the change instead of guessing at it.

## Give your agent the skill

Hermes keeps skills in `~/.hermes/skills/<name>/SKILL.md`. In your agent's session, send:

```
Create the file ~/.hermes/skills/pick-first-tool/SKILL.md with exactly the content below, then list your skills to confirm it loaded.
```

Paste the whole `SKILL.md` under that message, then start a new chat and try one of the requests from `tool-pick.csv`.

## Optional upgrades

- **A custom metric**, if Ground Truth Faithfulness gives partial scores. Under **Custom metrics**, click **Add metric**:
  - Name: `Right tool`
  - Description: `Did the agent pick the same first tool as the expected response?`
  - Scoring prompt: `Compare the tool name in the output with the expected response. Score 1 if they name the same tool, ignoring case, punctuation and extra words. Otherwise score 0.`
  - Tick **Requires ground truth**, and make `Right tool` the Star Metric with the pass threshold at the maximum score.
- **Test a router:** set the candidate type to **Model Router**, pick your router, and run the `with-skill` setup. Compare its score, cost and latency with the single model.
- **Write your own skill:** copy `skills/pick-first-tool/`, rewrite the rules for a task from your own work, and replace the CSV rows with real requests and their right answers.
