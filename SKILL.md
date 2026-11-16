---
name: noisy-reader
description: Keep the main thread's context small by handing noisy reads to a cheap reader model and keeping only its short, cited answer. Use when a read is likely to return more than about 5K characters and you need only a fact or excerpt from it, or when you would make three or more reads in a row from one source - CI and build logs, ticket and wiki payloads, large files or diffs, broad searches, API dumps. Routes to Haiku first, then a low-cost Gemini through GitHub Copilot, then a low-effort OpenAI model, whichever is installed locally, with tools switched off and the data treated as untrusted. Not for writes or mutations, text you will edit or quote exactly, or diagnosis that spans several sources. Never invoke from inside a NOISY_READER_CHILD or RUBBER_DUCK_CHILD run.
---

# Noisy Reader 📖

A big log or ticket dump read straight into the main thread costs context
for the rest of the task, at the main model's price, to get one fact. This skill
sends that read to a cheap model and brings back a short answer with verbatim,
line-cited evidence.

## Guard against recursion

If `NOISY_READER_CHILD=1` or `RUBBER_DUCK_CHILD=1` is set in the environment or
the prompt, do not use this skill. Answer from the data you were given.

## When to use it

Delegate when either is true:

- the read is likely to return more than about 5K characters and you need
  only a fact or an excerpt from it;
- you would make three or more reads in a row from one source (CI jobs and
  logs, tickets, wiki pages, big files, diffs, broad searches).

Keep these in the main thread:

- writes and mutations of any kind, such as triggering builds or editing
  tickets;
- content you will edit, or quote exactly in something you publish;
- small reads, where delegation costs more than it saves;
- diagnosis that has to join several sources (logs plus code plus tickets).
  Use the reader for each source's facts, and do the joining yourself.

## Pick the path

| The data comes from | Use |
|---|---|
| A shell command, a file, or anything you can save to a file | **The router**, below |
| An MCP server or another tool whose output you cannot redirect to a file | **A cheap subagent**, below, if the harness can choose a subagent's model |

### The router

1. Write the raw output to a file **without printing it**, in a private
   per-user directory:

   ```bash
   dir="${NOISY_READER_DIR:-${TMPDIR:-/tmp}/noisy-reader-$(id -u)}"
   mkdir -m 700 -p "$dir" && <command> > "$dir/<name>.txt" 2>&1
   ```

   Or pipe it straight in with `--save-as <name>`. The router saves it in the
   same directory, as a new file that never overwrites an existing one.
2. Ask one specific question, naming the job, build, ticket or path:

```bash
python3 <skill-dir>/scripts/noisy_reader.py \
  --question "Which stage failed in build 812, and what was the first error?" \
  --file "$dir/build-812.txt" \
  --grep 'ERROR|FAIL|Exception' --context 3
```

- `--grep` and `--context`, and either `--head` or `--tail`, narrow each source
  before it is sent, and keep the original line numbers so the evidence stays
  citable. Two files with the same name are labelled by their full paths.
- Input over 400,000 characters after narrowing is refused with exit code 3,
  as is input too large for every installed reader. Narrow it further or
  split the question by source; do not raise the cap to force it through.
- `--family anthropic|google|openai` or `--reader <cli>` forces a route.
  `--check` reports what is installed without a model call.
- `--format json` returns the reader, model, effort, answer and raw paths.

The router tries routes in order and stops at the first answer:

| Order | Model | Through |
|---|---|---|
| 1 | Haiku, effort low | `claude`, then GitHub Copilot |
| 2 | Gemini Flash | GitHub Copilot |
| 3 | GPT Luna, effort low | `codex`, then GitHub Copilot |

An unknown or plan-limited model steps down to the next model on the same
CLI. A missing, signed-out or throttled CLI is skipped for every family it
serves. Models live in [`data/readers.json`](data/readers.json); xAI Grok
models are never routed to.

Every route runs the reader without tools, so text in the data cannot make it
read, run or change anything:

| CLI | How tools are kept off |
|---|---|
| `claude` | `--tools ""`, plan permission mode, no session persistence |
| `copilot` | an empty `--available-tools=` set, built-in MCP servers disabled, no custom instructions |
| `codex` | every tool feature the installed CLI lists (`shell_tool`, `unified_exec`, browser and computer use, apps, plugins, multi-agent and others) is disabled, in a read-only sandbox without user config. A CLI whose `shell_tool` cannot be disabled is skipped. |

The data sits inside the prompt between random delimiters and is marked as
untrusted. A CLI with no verified way to run without tools, such as `agy`, is
not a route.

### A cheap subagent

When the source is only reachable through an MCP server or a harness tool,
spawn a subagent on the harness's cheapest capable model and give it the brief
in [`references/brief.md`](references/brief.md):

- **Claude Code:** an `Agent` call with `model: "haiku"`. The task is
  `Follow <skill-dir>/references/brief.md. Question: <specific question, with
  job or build, ticket key, path or repo>`.
- **Other harnesses:** use their subagent mechanism only when it can pin a
  cheap model. Otherwise fetch through a CLI the router can read (most CI and
  ticket systems have one), or narrow the read yourself.

This path is different in kind from the router: the subagent has to fetch, so
it keeps the parent's tools, and untrusted data reaches a model that can call
them. The brief's read-only rules and the harness's own permission prompts
are what keep it safe, not an empty tool set. So:

- Use it only when the router cannot be used, that is, when the data is only
  reachable through an MCP server or a harness tool.
- Never give it a task that needs a write, and never run it with permission
  checks bypassed.
- When the harness lets you define a subagent with a restricted tool list
  (for example a Claude Code agent definition limited to read tools), prefer
  that over a general-purpose subagent.

## Use the answer

- Report which reader answered: copy the router's `📖 Reader:` line, or name
  the subagent's model.
- Before acting on a fact, or quoting it in anything published, check the
  cited lines in the raw file with a narrow read (`sed -n '<from>,<to>p'`,
  `rg -n`). The reader is cheap, not authoritative.
- When the answer says the data does not answer the question, believe it. Ask
  a better question or fetch a different source rather than re-asking.

## When every route fails

The router exits 1 and lists each skipped route with its reason. Do the read
yourself, narrowly: search before reading whole logs, project only the fields
you need, and read slices with offset and limit.

## Credits

The brief and the routing rules are adapted from Ali Abdelhamid's
`noisy-reader` brief, and the router follows `steves-rubber-duck`'s routing and
fallback design. Both are MIT; see [`CREDITS.md`](CREDITS.md).
