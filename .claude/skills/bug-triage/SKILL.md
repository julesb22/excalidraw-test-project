---
name: bug-triage
description: Triages a bug end to end. It runs the bug-reproducer subagent to prove the bug with a failing test, then, only if the bug is reproduced, hands its report to the bug-analyst subagent, which gives the root cause, size and a GO, LATER or NO-GO recommendation, and files a ticket on the Notion bug board when one is given. Use it when someone reports a bug in any form (a GitHub bot message in a Slack thread, a plain description, a chat or Slack message, an issue URL) and wants it reproduced, analyzed or triaged. A bug description alone is enough to start.
argument-hint: [bug description | issue URL] [slack: <channel> <thread_ts>] [board: <Notion bug board link>]
---

# Bug triage

You run two expert subagents one after the other, and pass the first one's report to the second. You don't reproduce or analyze the bug yourself, and you never fix it.

## Input

The only thing you need is **the bug**. Find it in this order:

1. **A Slack thread** (the usual case). If the input gives a Slack channel and thread timestamp, read the thread with `slack_read_thread`. The parent message is normally a GitHub bot card ("Issue created by …"): its text is empty and the content is in the attachment. Take the issue URL from the card.
2. **A GitHub issue** (URL, or `owner/repo#N`), from the card or from `$ARGUMENTS`. Fetch its full title and body with the GitHub MCP `issue_read` tool: the Slack card can truncate the body, and there is no `gh` CLI here. The bug is the issue's title and body, as written, plus its URL.
3. **Plain text** in `$ARGUMENTS`, or the message that called you, if there is no thread and no issue. Use the reporter's words.

An empty `Bug:` field does not mean there is no bug: check the thread first. If the thread's parent is not a GitHub card, use its text as the bug.

A description alone is enough: start right away. Only if all three come up empty, stop and say that no bug description was found. Don't ask a question when running unattended: nobody is there to answer.

**The Notion bug board is optional.** Look for its link, in this order:
1. after `board:` in `$ARGUMENTS`, or any Notion link given with the bug;
2. a line starting with `Bug board:` in the project's `CLAUDE.md`.

If you find none, don't ask for one and don't stop: run the whole triage without it. The analyst then returns its analysis in its answer instead of filing a ticket.

## Step 1: reproduce

Call the **bug-reproducer** subagent. Give it:
- the bug, exactly as you found it: for a GitHub issue, its URL plus its title and body as is (so the reproducer doesn't need to fetch it again); otherwise the reporter's text;
- the project root: the current working directory.

Don't rephrase the bug and don't add your own guesses about its cause.

Then read the first line of its report, the verdict:

| Verdict | What you do next |
|---|---|
| `yes` | Go to step 2. |
| `no`, `stuck` or `needs info` | Stop and hand back to the human. Don't call the analyst: no ticket is filed for a bug that isn't reproduced. Start your answer with **I can't reproduce this bug.**, then show the report, then ask the reporter for what would help: the missing information for `needs info`, or the details that could make it reproducible for `no` or `stuck` (exact steps, browser, file, screenshot). |

## Step 2: analyze and file

Call the **bug-analyst** subagent. Give it, in this order:
1. the Notion bug board link, or "No board: return the analysis in your answer" if you have none;
2. the original bug, exactly as you passed it to the reproducer, including the issue URL when there is one, so the ticket links back to the GitHub issue;
3. the bug-reproducer's report, **copied as is**, from its first line to its last. Don't summarize it, reformat it, or fix anything in it: the analyst copies its steps into the ticket, so any change you make ends up there.

## Output

Show the user both reports, unchanged, with nothing before them:

```markdown
## 🕵️ bug-reproducer

<the bug-reproducer's report, as is>

## 🧠 bug-analyst

<the bug-analyst's answer, as is>
```

If you stopped after step 1, the answer is: the line **I can't reproduce this bug.**, then only the first part, then your question to the reporter.

After the two reports, add at most one line, only if something needs the user's attention: for example, a subagent failed, the analyst left a property empty, it updated an existing ticket instead of creating one, or no ticket was filed (say why: no board link, or Notion isn't connected here).

## Rules

- Never fix the bug and never edit application code. The only project file the run leaves behind is the bug-reproducer's test, when the bug is reproduced.
- Don't commit or push anything.
- If a subagent can't be found, say so and stop: it may need a new session to be loaded.
