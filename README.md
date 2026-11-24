# spec-format

What a written spec contains, and how its fresh-eyes review runs, kept in one place for the [Claude Code](https://docs.claude.com/en/docs/claude-code) flows that write specs.

- **`format.md`** — what a spec contains: where it lives and its name, length, the template, why Outcome, Key decisions and Open Question defaults matter, a worked example, how to call the implementation strategy, and style.
- **`review.md`** — the fresh-eyes review: starting one, what the reviewer looks for and returns, when a review counts, and handling its findings.

## Not a skill

There's deliberately no `SKILL.md`, so Claude Code skips this folder: it never appears in the skill list and never triggers on its own. Nobody types it; the flows read it.

## Who reads it

- [`/spec`](https://github.com/IngvarKofoed/claude-code-spec) reads both files.
- [`/feature-flow`](https://github.com/IngvarKofoed/claude-code-feature-flow) reads both when it designs.

Each looks for the `spec-format` folder beside its own skill folder and stops, with the install command below, if either file is missing.

## Install

```
/skill-manager:add https://github.com/IngvarKofoed/claude-code-spec-format.git --dir spec-format
```

That puts it at `~/.claude/skills/spec-format/` and keeps it updated like the other skills.

## The contract

Both flows rely on these names, so renaming any of them is a change to both flows:

- the files `format.md` and `review.md`;
- `review.md`'s four part headings: **Starting a review**, **For the reviewer**, **When a review counts**, **Handling the findings**.

The files never mention either flow's steps or skills. When things happen is each flow's business; these files say what a spec contains and how a review sorts its findings.
