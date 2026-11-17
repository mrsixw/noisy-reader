# noisy-reader 📖

An agent skill that keeps a coding agent's context small. Instead of reading a
40,000-line build log or a ticket dump into the main thread to find one fact,
the agent hands the read to a cheap model and keeps only a short answer with
verbatim, line-cited evidence.

- **Router:** `scripts/noisy_reader.py` sends the data, narrowed and
  line-numbered, to the first reader that works: Haiku through `claude` or
  GitHub Copilot, then Gemini Flash through Copilot, then a
  low-effort GPT through `codex` or Copilot. Every route runs the reader with
  its tools switched off, and the data is marked as untrusted.
- **Subagent brief:** `references/brief.md`, for sources only an MCP server
  can reach, run on a cheap subagent model such as Claude Code's `haiku`.

See [`SKILL.md`](SKILL.md) for when to use it and how.

## Install

```bash
git clone https://github.com/mrsixw/noisy-reader.git "$HOME/.agents/skills/noisy-reader"
for agent in claude codex copilot; do
  [ -d "$HOME/.$agent" ] && mkdir -p "$HOME/.$agent/skills" &&
    ln -s "$HOME/.agents/skills/noisy-reader" "$HOME/.$agent/skills/noisy-reader"
done
```

Update with `git -C "$HOME/.agents/skills/noisy-reader" pull --ff-only`.

## Example

```bash
dir="${TMPDIR:-/tmp}/noisy-reader-$(id -u)"; mkdir -m 700 -p "$dir"
make test > "$dir/make-test.txt" 2>&1
python3 scripts/noisy_reader.py \
  --question "Which tests failed, and with what assertion?" \
  --file "$dir/make-test.txt" --grep 'FAIL|Error' --context 2
```

```text
📖 Reader: claude (haiku, effort low), anthropic

**Answer:** Two tests failed, both on missing strings in the skill text.
**Evidence:**
- `make-test.txt:5` FAIL: test_requires_exact_draft_and_head_sha_approval
- `make-test.txt:14` FAIL: test_simplicity_lens_is_advisory_only

**Raw:** /tmp/noisy-reader-501/make-test.txt
```

## Requirements

Python 3.10 or later, standard library only, and at least one of
[Claude Code](https://code.claude.com/docs/en/headless),
[GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/copilot-cli)
or [Codex CLI](https://developers.openai.com/codex/cli), signed in.
`--check` shows which are ready.

## Development

```bash
python3 -m unittest discover -s tests -v
python3 -m py_compile scripts/noisy_reader.py
python3 scripts/noisy_reader.py --check
```

Models live in [`data/readers.json`](data/readers.json). Keep a superseded
model in its route as a fallback rather than deleting it.

Do not add xAI Grok models, even where a CLI such as Copilot offers them. The
maintainer strongly prefers other families, so the catalogue leaves Grok out
on purpose. A test already checks the catalogue for it.

## Credits

Adapted from Ali Abdelhamid's noisy-reader brief (MIT), with routing that
follows [`steves-rubber-duck`](https://github.com/mrsixw/steves-rubber-duck).
See [`CREDITS.md`](CREDITS.md).

## Licence

[MIT](LICENSE)
