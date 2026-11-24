# What a spec contains

A spec is the design written down before any code changes happen: where it lives, how long it runs, which sections it carries and why, and how it calls its own implementation strategy. Follow this file every time you draft or revise one.

## Where a spec lives

Write the spec to:

```
<repo-root>/docs/specs/YYYY-MM-DD-<slug>.md
```

- `YYYY-MM-DD` is today's date. Use `date +%Y-%m-%d` if unsure; the system date in your context is also fine.
- `<slug>` is short, kebab-case, descriptive: `treasure-loot-tables`, `path-capacity-upgrades`, `session-rotation`. Aim for 2–4 words.
- Create `docs/specs/` if it doesn't exist.
- If the project's existing conventions clearly call for a different location (e.g., a `CLAUDE.md` says specs live in `requirements/`, or the user explicitly names a path), follow that instead.

## Length calibration

Most specs are **50–200 lines**. Past 300 lines, you're either tackling something genuinely large or you're padding — and the model's bias is strongly toward padding. A reviewer who can't hold the whole spec in their head can't catch architectural problems in it. Resist the urge to include every type definition, every route signature, every migration step. The spec is the *thinking*, not the *implementation*.

If you find yourself approaching 300 lines, ask: is this one spec, or three? Splitting a large feature into multiple smaller specs is usually right.

(The Outcome section and the Implementation strategy section don't count against this budget. They're a handful of lines each, and they aren't design.)

## Template

Use this as a starting skeleton. Adapt section names and depth — don't pad with empty sections, and add new ones only when they earn their keep.

```markdown
# <Feature name>

<One-paragraph summary: what this feature is, why it exists, and the
chosen approach in one breath.>

## Outcome

<Written with the draft and kept in sync with the Design on every
pass, so the user signs off on it. The reader's fast answer to "what
do I get, and how do I know it works?" Two short bullet lists, 2–4 bullets each. Keep both
observable from outside the code — capabilities and checks, not
implementation steps. This is not a test plan: each verify bullet is
an action anyone can take plus the result they should see, not a
test matrix.>

**What you get:**

- <Concrete, user-visible capability or change that exists once this
  ships>

**How to verify:**

- <An action and the observable result that confirms the
  implementation matches the spec>

## Key decisions

<The load-bearing choices a reviewer should check before approving.
This is the section a code reviewer reads first — surface them here,
don't bury them in Design. Cover both kinds:

- **Code-shape choices** — which existing pattern to reuse, where a
  check lives, which abstraction owns a piece of state.
- **Tech-stack choices** — which package or library to add, which
  dependency to drop, which framework / runtime / build tool to
  pull in. A wrong dependency is just as load-bearing as a wrong
  pattern.

Each entry is one short bullet tagged with how it relates to the
existing codebase:

- `(reuses)`   — adopts an existing pattern as-is
- `(extends)`  — reuses with deliberate additions or modifications
- `(new)`      — introduces a pattern not previously in the codebase
- `(diverges)` — goes against an existing convention, deliberately
- `(breaking)` — changes a contract other code depends on

Aim for 4–8 entries. More than 10 usually means you're listing
implementation details instead of decisions.>

- **<Short decision name>** (tag). <One- or two-sentence description
  of what was decided and why, especially the relationship to
  existing code.>

## Goals

- <What this is trying to accomplish>

## Non-goals

- <What it explicitly is not trying to do — bounds the discussion>

## Design

<The meat. Structure is flexible. Use whatever subsections make the
design legible. Reference real file paths and identifiers where it
helps. Code snippets are welcome when they pin down a tricky bit, but
the spec is prose, not an implementation.>

## Open questions

- **Q:** <Specific, narrow, answerable question.> **Default:** <Your
  best-guess answer. Stating a default lets the user accept by
  silence rather than having to type a reply for every question.>

## Alternatives considered

<Only if approaches were discussed and rejected. Two or three lines
each is plenty — the point is not to re-litigate, it's to record why
this path was taken.>

## Implementation strategy

<Written with the draft and kept in sync on every pass. Strategy,
agent count, and model, in 3–6 lines. See "Calling the
implementation strategy" below.>
```

## Why Outcome is in the draft

The summary says what the feature is; Outcome says what you can point at once it exists. "What you get" is the deliverable in user-visible terms, and "How to verify" is the acceptance check — action plus expected observation. Together they give whoever builds the spec a finish line that isn't "the diff looks plausible."

It's *placed* at the top because it's the first thing a reader of the finished spec needs. It's in the draft from the start because whoever builds the spec is graded against it, so the user must sign off on it, not just on the Design. It's derived from the design, which means it moves when the design moves. Re-derive it whenever the Design changes, and never let a stale bullet contradict the design the user is about to approve. Don't recycle the Goals section: Goals state intent, Outcome states what you can point at and check. Keep the verify bullets black-box — an action anyone can take plus the observable result. The moment they name test files or frameworks, they've become the Testing section the spec deliberately doesn't contain.

## Why "Key decisions" matters

A reviewer's job is to catch architectural mistakes — the spec inventing a new pattern when an existing one would do, or diverging from a convention without good reason. With the load-bearing choices buried in Design, that work means reading the whole spec. With them surfaced at the top, tagged by their relationship to the existing codebase, a reviewer can scan the `(new)` and `(diverges)` bullets in under a minute and spend the rest of their attention on the genuinely novel parts.

This is the section that does the most work per line of any in the spec. Treat it as required, not optional.

## Why Open Questions take defaults

Stating `Default: X` after every question lets the user accept by silence. Without defaults, every question is a blocker — the user has to type something on each one before the spec moves forward. With defaults, the spec is shippable as-is, and the user only weighs in on the questions where the default is wrong.

## A worked example (right-sized)

A complete spec for a small feature, demonstrating the shape and density that "right-sized" means in practice. ~70 lines, all sections present, none padded.

````markdown
# Skip inactive users in the daily digest

Add a guard to the daily-digest job so users who haven't logged in
for 30 days don't receive the email. The cutoff is checked at send
time against `users.lastLoginAt`; users below the threshold are
silently skipped.

## Outcome

**What you get:**

- Users with no login in the last 30 days stop receiving the
  daily digest.
- A daily metric of how many users were skipped for inactivity.

**How to verify:**

- Set a test user's `lastLoginAt` 31+ days back, run the digest
  job: no email arrives and no `digest_sends` row is written.
- The same run increments `digest.skipped_inactive` by one.
- A user who logged in yesterday still receives the digest.

## Key decisions

- **Where the check lives** (extends). Runs inside
  `DigestJob.shouldSend(user)` next to the existing unsubscribe
  check, not at the recipient-query level. Reason: keeps all
  "should this user receive a digest" logic in one place.
- **Threshold value** (new). 30 days, hardcoded as
  `INACTIVE_DAYS = 30` in `digest.ts`. Not a config — we don't
  expect tuning, and a constant is easier to grep.
- **Send-log behavior** (diverges). Skipped users get no row in
  `digest_sends`. Existing convention logs every attempted send;
  we diverge because "didn't try" and "tried and suppressed" are
  meaningfully different states.
- **`date-fns` dependency** (reuses). The 30-day arithmetic uses
  `differenceInDays` from `date-fns`, already a project dep.
  Considered raw `Date` math; rejected for readability.

## Goals

- Stop emailing users who have effectively churned.
- Have the skip be observable in metrics (daily count of
  inactive-skipped users).

## Non-goals

- Re-engagement campaigns for churned users.
- A configurable threshold per workspace.

## Design

`DigestJob.run()` already iterates eligible users and calls
`shouldSend(user)`. Add a guard at the top of `shouldSend`:

```ts
const inactive =
  user.lastLoginAt == null ||
  differenceInDays(Date.now(), user.lastLoginAt) > INACTIVE_DAYS;
if (inactive) {
  metrics.inc("digest.skipped_inactive");
  return false;
}
```

Users with `lastLoginAt = NULL` (admin-created, never logged in)
are treated as inactive and skipped.

## Open questions

- **Q:** Should re-activation flip eligibility same-day, or wait
  for the next daily run? **Default:** wait for the next run — the
  user already gets a login-confirmation email, so a same-day
  digest would be noise.

## Alternatives considered

- **Filter at the recipient query** (`WHERE lastLoginAt > ...`).
  Cheaper but scatters the eligibility logic across the query and
  `shouldSend`. Rejected — one place beats two.
- **Soft-delete inactive users.** Out of scope; we may still need
  to email them for account purposes (password expiry, etc).

## Implementation strategy

*Not part of the design — a starting point for whoever builds this.*

- **Single agent, Opus 5.5.** Two files (`digest.ts` and its test), one
  code path, and the metrics counter has to land alongside the guard
  — there's no chunk a subagent could take without re-reading the rest.
````

Note what this example *doesn't* have: no Implementation Order section, no Testing section (the Outcome verify bullets are black-box acceptance checks, not a test plan), no Migration section, no Deployment section. Those belong in the build phase — they're not design decisions. The Outcome section and the Implementation strategy are the two deliberate exceptions — Outcome because it's the acceptance contract for the design, the strategy because it's a launch instruction rather than a re-derivable plan.

## Calling the implementation strategy

The spec says what to build. It also needs a starting point for *how* — which agent strategy, how many agents, which model. Someone will decide whether to run this in one session, fan it out, or throw an orchestrated workflow at it, and you've spent the whole spec phase reading the exact code it touches. Nobody is better placed to guess.

Every draft carries a short **Implementation strategy** section at its end, kept in sync on every pass, so the user signs off on it too. It goes in the file rather than staying in the conversation because the build often happens in a different session — a chat line is gone by then; a section is still there when someone opens the spec to start work.

### Strategy

One of three, picked on the shape of the work. Neither single agent nor multi-agent is the safe default; they fail in opposite directions. Fan out a tightly coupled change and the agents conflict and guess at each other's context. Keep a many-part build in one session and it grinds through a single context window, pays the lead's model rate for mechanical work, and has lost the early decisions to compaction by the end.

- **Single agent** — one session, working sequentially. The change is one coherent thread: a few files on one code path, each edit building on the last, nothing that could be handed off without first explaining most of the rest.
- **Multi-agent** — the implementer session leads and delegates bounded chunks to subagents. Usually it builds the core itself (the part that interprets a `(new)` or `(diverges)` decision) and hands off the rest; when the streams are fully independent, it may only coordinate and integrate. Chunks run in parallel when they don't touch each other's files, and one after another when one builds on the last; both count. A chunk qualifies when it has a settled input (an interface or decision that the spec, or the lead's first step, pins down), its own files, and a check for done. Typical chunks: one per adapter or module behind a shared interface; the call-site sweep after a signature change; the UI over a finished backend; tests written against the Outcome bullets. The test: *can you name one chunk a fresh agent could finish from the spec plus a one-paragraph brief?*
- **Ultracode** — orchestrated fan-out with adversarial verification. Reserved for when a wrong implementation is expensive and hard to walk back: a `(breaking)` or irreversible Key decision, a data-integrity or security surface, or correctness that can't be eyeballed from a diff. It costs a great deal of tokens, so it has to earn them.

Breadth is a signal of its own. A spec whose Design spans more than one layer or subsystem (backend and UI, a new module and its call sites, an interface and its implementations) almost always has a chunk that passes the multi-agent test. One that stays inside a single module rarely does.

The spec's own **Key decisions** tags are the cheapest signal available. A wall of `(reuses)` / `(extends)` on one code path is single-agent work. `(new)` across independent subsystems is a chunk per subsystem. A `(breaking)` signature change whose callers all live in this repo is a call-site sweep, a natural cheap-model chunk. A `(breaking)` contract other systems depend on, or a migration that can't be undone, is what pushes toward ultracode.

### Agent count

For a single agent, skip it. For multi-agent, the count *is* the number of chunks you can actually name. Say what each one owns, in real file paths, whether the chunks run in parallel or in sequence, and what the lead keeps for itself (the core, the integration pass, or both). One delegated chunk is a legitimate answer; two to four is typical. If you can't name what agent #4 owns without overlapping agent #2, there are three chunks, not four.

For ultracode, describe the shape rather than a headcount ("three finders, then adversarial verify per finding"). The harness caps how many agents run at once, so a bigger number buys queueing, not speed.

### Model

The principle outlasts the names: **pick the cheapest tier that clears the work's difficulty bar.** As of 2026-11, that means (prices in USD per million input / output tokens):

- **Opus 5.5** (4 / 20) — the default for implementation. Multi-file features, refactors, anything where the builder has to *interpret* a `(new)` or `(diverges)` decision rather than transcribe it.
- **Sonnet 5.5** (2 / 10) — close to Opus on coding and agentic work for half the price. The right default for delegated chunks in a multi-agent build, where the model cost is paid N times over.
- **Haiku 5.5** (0.10 / 0.50, rising to 0.50 / 2.50 for prompts past 100K tokens) — well-specified mechanical transforms with no judgment in them: renames, mass call-site updates, boilerplate.
- **Fable 5.2** (8 / 40, twice Opus) — the hardest, longest-horizon work: an overnight autonomous run, or a problem where it's genuinely unclear whether the approach is tractable. Reserve it for that.

Mixing tiers inside one strategy is normal and usually right — Opus on the stream carrying the design risk, Sonnet or Haiku on the mechanical ones. Say which agent gets which.

### The section

It sits below Alternatives considered. Keep it to 3–6 lines — it's a starting point, not a plan, and it doesn't count against the spec's length budget.

````markdown
## Implementation strategy

*Not part of the design — a starting point for whoever builds this.*

- **Single agent, Opus 5.5.** Four files on one code path, and the edits
  depend on each other — parallelism would only produce conflicts.
````

A multi-agent one, showing what each agent owns and why the tiers differ:

````markdown
## Implementation strategy

*Not part of the design — a starting point for whoever builds this.*

- **Multi-agent, 3 streams.** One agent per adapter —
  `adapters/stripe.ts`, `adapters/paypal.ts`, `adapters/adyen.ts` —
  which share only the `PaymentAdapter` interface. Then a single
  integration pass on `router.ts`.
- **Sonnet 5.5 for the adapters, Opus 5.5 for the integration pass.** The
  adapters transcribe a settled interface; the router is where the
  `(diverges)` retry decision has to be interpreted.
````

The more common multi-agent shape: the lead builds the core, then hands off what hangs off it:

````markdown
## Implementation strategy

*Not part of the design — a starting point for whoever builds this.*

- **Multi-agent: the lead plus 2 chunks.** The lead builds the `(new)`
  rotation core in `auth/session.ts` and settles `rotate()`'s
  signature. Then, in parallel: one subagent moves the 23 callers in
  `routes/` to `rotate()`, another builds the toggle in
  `web/settings/` against it.
- **Opus 5.5 lead, Haiku 5.5 for the call sites, Sonnet 5.5 for the
  toggle.** The sweep is mechanical; the toggle needs judgment but no
  design calls.
````

Its headline is one line long, for wherever the call is summarized to the user without the file:

> *"Implementation strategy: **single agent on Opus 5.5** — four files on one code path, and the edits depend on each other."*

**Call it in both directions.** Don't inflate: multi-agent on a four-file thread buys merge conflicts and agents guessing at each other's context, ultracode on a small change just burns tokens, and the top model tier by reflex is waste. If it's a single-agent Sonnet job, say so. But don't shrink by reflex either. In practice the pull toward "single agent" is the stronger one, because it feels safe, and it isn't free: a long build in one context gets slower, costlier and less sharp as it goes. If you can name a chunk that passes the multi-agent test, call multi-agent.

**This one appendix is not a door for the others.** Implementation order, test matrices, deployment runbooks, and dependency lists still don't belong in the spec. The strategy section earns its place because it's a launch instruction the next session can't re-derive from the design — everything on that list can be.

## Style guidelines for the spec itself

- **Concrete over abstract.** "Adds a `carrierLimit: number` field on `Path`, default 1" beats "introduces capacity tracking on paths."
- **Reference real names.** File paths, function names, types — they make the spec auditable against the code.
- **Explain *why*, not just *what*.** A reader six months from now needs to know the reasoning, not just the rule. The why is the part the code alone can't capture.
- **Right-size.** See the length calibration above. Don't include a section heading if there's nothing meaningful under it.
- **Match the project's existing voice.** If the project's docs are narrative prose, write narrative prose. If they're bullet-heavy, lean bulletier. Skim a sibling doc before writing.

## Common mistakes to avoid

- **Treating implementation logistics as design.** Implementation order, test matrices, deployment runbooks, NuGet / dependency change lists — these belong in the build phase, not the spec. If you find yourself writing a "Testing" or "Implementation order" section, ask whether it's making any genuine *design* decision. If not, cut it. (The Outcome section's "How to verify" bullets are the deliberate exception: black-box acceptance checks, not a test plan — the moment they name test files or frameworks, they've crossed the line.)
- **Burying the architectural choices in Design.** A reviewer should be able to find the load-bearing decisions in the Key Decisions section without reading the rest. If the `(new)` and `(diverges)` bullets aren't there, you've made the reviewer's job much harder than it needs to be.
- **Skipping the Open Questions section.** It's the cheapest way to surface "I'm not sure about this" without blocking the draft. Use it generously, with `Default: X` for each entry.
- **Letting Outcome or the strategy go stale between passes.** Both are derived from the design, and the user signs off on both. A pass that changes the Design without bringing them along hands the user an acceptance contract that contradicts what they're approving. Re-derive them on every pass, and say in a line when either one moved.
- **Calling the strategy by reflex.** The strategy call picks on the shape of the work, not a default. Multi-agent on a tightly coupled thread buys merge conflicts; single agent on a build with a nameable, self-contained chunk leaves the lead grinding through one context at its own model rate. Ultracode still needs a genuinely expensive-to-undo change, and the model is still the cheapest tier that clears the bar.
- **Letting the strategy appendix grow into a plan.** It's 3–6 lines: strategy, agent count, model, and why. The moment it starts listing implementation order or test cases, it's become the build plan the spec deliberately doesn't contain — and it drags the other logistics sections back in with it.
