# Credits

## noisy-reader brief, by Ali Abdelhamid

- **Source:** `claude/briefs/noisy-reader.md` and the "Haiku 5.5: noisy reads"
  section of `claude/CLAUDE.md` in Ali Abdelhamid's public `agent-skills`
  repository on GitHub, at commit `3f4f512`.
- **Author:** Ali Abdelhamid.
- **Licence:** MIT, reproduced below.

### What was adopted

- Delegating a read when it is likely to return more than about 5K characters
  and only a fact or excerpt is needed, or when three or more reads in a row
  hit one source; keeping writes, exactly quoted content, small calls and
  multi-source diagnosis in the main thread.
- The subagent brief: answer only the question, do the work yourself,
  read-only, narrow before reading, save raw output under
  `/tmp/noisy-reader/`, say what was checked instead of guessing, and the
  Answer, Evidence and Raw reply under about 1,500 characters.

### What was changed

- **A local router** (`scripts/noisy_reader.py`) replaces a fixed Haiku
  subagent. It falls back from Haiku to a low-cost Gemini and then to a
  low-effort OpenAI model through whichever CLIs are installed, runs the reader
  with no tools, and treats the data as untrusted between random delimiters.
- **Narrowing in the router**: `--grep`, `--context`, `--head` and `--tail`,
  with original line numbers kept for citations, and a size cap that refuses
  rather than truncates.
- **The brief's tool list is generic.** The original named specific CI and
  ticket MCP tools; this version describes write operations by kind so that it
  applies to any server.
- **Recursion markers** (`NOISY_READER_CHILD`, `RUBBER_DUCK_CHILD`) are set on
  every reader process and honoured on entry.

```text
MIT License

Copyright (c) 2026 Ali Abdelhamid

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## steves-rubber-duck

The router's model fallback (step down on an unknown or plan-limited model,
skip a throttled or signed-out CLI), its error patterns, its tool-less CLI
invocations and its recursion marker follow
[`mrsixw/steves-rubber-duck`](https://github.com/mrsixw/steves-rubber-duck),
by the same author and under the same MIT licence as this repository.
