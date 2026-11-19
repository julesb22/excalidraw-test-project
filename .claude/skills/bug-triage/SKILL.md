---
name: bug-triage
description: Triages a bug end to end. It runs the bug-reproducer subagent to prove the bug with a failing test, then, only if the bug is reproduced, hands its report to the bug-analyst subagent, which gives the root cause, size and a GO, LATER or NO-GO recommendation, and files a ticket on the Notion bug board when one is given. Use it when someone reports a bug in any form (a plain description, a chat or Slack message, an issue URL) and wants it reproduced, analyzed or triaged. A bug description alone is enough to start.
argument-hint: <bug description or issue URL> [board: <Notion bug board link>]
---

# Bug triage

You run two expert subagents one after the other, and pass the first one's report to the second. You don't reproduce or analyze the bug yourself, and you never fix it.

## Input

The only thing you need is **the bug**. Take it in whatever form it comes:
- text in `$ARGUMENTS`: a description in plain English, or the URL of an issue (GitHub or another tracker);
- if `$ARGUMENTS` is empty or doesn't describe a bug, the bug described in the conversation that called you: the user's message, or the chat or Slack thread you were mentioned in. Use the reporter's words.

A description alone is enough: start right away. Ask a question only if you can't find any bug description at all.

**The Notion bug board is optional.** Look for its link, in this order:
1. after `board:` in `$ARGUMENTS`, or any Notion link given with the bug;
2. a line starting with `Bug board:` in the project's `CLAUDE.md`.

If you find none, don't ask for one and don't stop: run the whole triage without it. The analyst then returns its analysis in its answer instead of filing a ticket.

**The Slack thread is optional too.** It's given with the bug as two lines, `channel_id: C…` and `thread_ts: 1234567890.123456`, or as a Slack message link. Remove those two lines from the bug before passing it on: the subagents get only the reporter's words.

## Step 1: reproduce

Call the **bug-reproducer** subagent. Give it:
- the bug, exactly as the user gave it (the text, or the issue URL);
- the project root: the current working directory.

Don't rephrase the bug and don't add your own guesses about its cause.

Then read the first line of its report, the verdict:

| Verdict | What you do next |
|---|---|
| `yes` | Go to step 2. |
| `no`, `stuck` or `needs info` | Stop and hand back to the human. Don't call the analyst: no ticket is filed for a bug that isn't reproduced. Start your answer with **I can't reproduce this bug.**, then show the report, then ask the reporter for what would help: the missing information for `needs info`, or the details that could make it reproducible for `no` or `stuck` (exact steps, browser, file, screenshot). Then go to step 3. |

## Step 2: analyze and file

Call the **bug-analyst** subagent. Give it, in this order:
1. the Notion bug board link, or "No board: return the analysis in your answer" if you have none;
2. the original bug, exactly as the user gave it;
3. the bug-reproducer's report, **copied as is**, from its first line to its last. Don't summarize it, reformat it, or fix anything in it: the analyst copies its steps into the ticket, so any change you make ends up there.

## Step 3: reply in Slack

Only if a Slack thread was given. Post one reply in that thread with Slack's send-message tool: `channel_id` and `thread_ts` from the input, no broadcast. Post nowhere else.

- Reproduced: one line each for the root cause, the size, the recommendation and the Notion ticket link.
- Not reproduced: **I can't reproduce this bug.**, then your questions for the reporter.

End the reply with a link back to this session so the reader can answer there. Build it from the `CLAUDE_CODE_REMOTE_SESSION_ID` environment variable: swap its `cse_` prefix for `session_` and append it to `https://claude.ai/code/`. Write it as: `👉 Reply "go ahead" or "no" here: <link>`. If the variable isn't set, leave the link out.

If no thread was given, or Slack isn't connected, skip this step.

## Output

Show the user both reports, unchanged, with nothing before them:

```markdown
## 🕵️ bug-reproducer

<the bug-reproducer's report, as is>

## 🧠 bug-analyst

<the bug-analyst's answer, as is>
```

If you stopped after step 1, the answer is: the line **I can't reproduce this bug.**, then only the first part, then your question to the reporter.

After the two reports, add at most one line, only if something needs the user's attention: for example, a subagent failed, the analyst left a property empty, it updated an existing ticket instead of creating one, or no ticket was filed (say why: no board link, or Notion isn't connected here), or the Slack reply wasn't posted (say why).

## Rules

- Never fix the bug and never edit application code. The only project file the run leaves behind is the bug-reproducer's test, when the bug is reproduced.
- Don't commit or push anything.
- If a subagent can't be found, say so and stop: it may need a new session to be loaded.
