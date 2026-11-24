# The fresh-eyes review

A review hands the spec the user is about to sign to a reader who wasn't in the conversation that produced it. This file has four parts: starting one, what the reviewer does, when a review counts, and handling what it finds.

## Starting a review

The spec is ready to sign — but the person who just wrote it is the worst-placed to judge it. By the time you've drafted and revised a spec, every ambiguity in it has already been resolved *in your head*; you read the words and see what you meant, not what they actually say. So hand the text the user is signing to a reader who wasn't in the room. It runs in the background, so it can overlap with the user reading the final text.

Spawn a subagent as a fresh reviewer and give it **only two paths**: the spec file, and this file with the instruction to follow its "For the reviewer" part. Nothing from the conversation — not the approach discussion, not the options you narrowed down, not the user's answers. That blindness is the whole point: a reviewer carrying your context inherits your interpretation and trips on nothing. Let it read the referenced code and docs read-only — it needs them to judge whether the design reuses what's there or reinvents it — but keep the conversation out of its head. This file carries no conversation, so pointing the reviewer at it keeps it blind. The brief, in full:

> *"Review the spec at `<spec path>`. Read `<path to this review.md>` and follow its "For the reviewer" part."*

Launch it on Fable (`model: "fable"` on the Agent call) and leave its effort at the default. Speed matters here because the user may be waiting on it. Over a month of runs, Fable 5.2 reviews took a median 2.8 minutes against 6.9 on Opus 5.5, with similar findings.

As it starts, note the spec's hash — see "When a review counts".

## For the reviewer

You're reading this spec cold, and that's the point: you weren't in the conversation that produced it, so you see what the words actually say rather than what the author meant. Read the spec, and read the code and docs it refers to — read-only — to judge whether the design reuses what's there or reinvents it. Don't edit anything; return what you found.

Look for six kinds of problem — all high-level, design-level concerns a fresh reader is uniquely placed to catch, and exactly the kind of thing the user should get a say in. They fall into three pairs: is it **clear** (ambiguity, completeness), is it **right-sized** (unnecessary complexity, goal-fit), and can you **live with it** (bad UX, reversibility).

- **Ambiguity** — a load-bearing part of the design that a competent implementer could reasonably read two ways and build differently. The test: *would two engineers implement this and both believe they followed the spec, yet ship incompatible things?* "This sentence could be tighter" is a nit; "it's unspecified whether the check runs before or after the write, and the data model differs depending" clears the bar.
- **Completeness** — the design covers the happy path but leaves a load-bearing case unsaid: failure, empty or malformed input, concurrency, or the existing data and clients at rollout. Where ambiguity is something said two ways, this is something not said at all — and the cold reader is the one most likely to notice, because the author already filled the gap in their head. The test: *is there a case a builder will hit and have to guess at, where guessing wrong changes the result?* Let the trivial omissions go; flag the ones that force a blind decision.
- **Unnecessary complexity** — machinery the spec introduces that isn't earning its keep: a new table, service, abstraction, or dependency where something simpler or already-present would do, or scope that has crept past the smallest interesting version of the feature. The test: *can you name a materially simpler design that still meets the stated Goals?* If yes, that's the finding.
- **Goal-fit** — the design doesn't actually deliver a stated Goal, or spends effort on things no Goal asked for. The test: *walk each Goal and point at the part of the Design that satisfies it, then walk the Design and check each piece traces back to a Goal.* A Goal with nothing behind it is a gap; a slab of Design with no Goal behind it is either a missing Goal or scope to cut (which is really the complexity finding wearing a different hat). The Outcome bullets are cheap anchors for this walk — check that they trace to the Design too.
- **Bad UX** — the experience of whoever *uses the thing being specced* (end user or operator) would be confusing, surprising, or unrecoverable as designed: a destructive action with no confirmation, a silent failure, an error with no path forward, a default that will surprise most users. This is the UX of the feature — not the UX of writing or reviewing the spec.
- **Reversibility** — the design commits to something expensive or impossible to undo: an irreversible migration, a breaking API or schema change, deleting data, a dependency that's hard to back out. This lens matters most here, at spec-time, because a one-way door costs almost nothing to reconsider now and a great deal once the branch exists. The test: *if this proves wrong after it ships, how hard is it to walk back?* The point isn't to veto it — it's to be sure the user chose the door knowingly, so it's always a decision, never a silent fix.

**What to return.** For each finding:

- **the category** — which of the six lenses it falls under;
- **where** — the section or line of the spec;
- **what's at stake** — what goes wrong if it stays as written;
- **the options** — the plausible resolutions, or the one fix if there's only one;
- **decision or fix** — a *decision* if more than one resolution is plausible, a *fix* if there's one obviously-correct resolution and no real tradeoff (a snippet that contradicts a stated decision, a missing null guard, two sections that disagree, an off-by-one in an example).

Raise a decision only at critical or high: something the user would genuinely want to decide before walking away. Leave lesser judgment calls out — a pile of things to adjudicate trains the user to skip the review. You can't know what the user already settled, so report what the text says; the session that launched you sorts that out. If nothing clears the bar, say so explicitly in one line — an empty reply reads as a failed review.

## When a review counts

A review vouches only for the exact text it read. Note the spec's hash (`shasum <spec>`) when it starts. That's how you'll tell later whether the text it read is still the text being signed. Whenever the spec may have changed — the user can edit it at any time, including while reading — re-read it from disk and compare the hash.

**Stale means the text changed.** Once the spec changes after a review started, that review no longer vouches for it, whether it's still running or already back. The change can come from the user's edits or from a revision you make. A stale review's findings don't count for the new text; a fresh review has to read it. Two kinds of edit don't count: applying that review's own fixes, and folding in Open Question answers that only accept the stated defaults, which the review already read.

**A failed review is no review.** A review that fails, returns nothing, or is lost counts as no review — never as "nothing to decide". Say so, re-run it once, and if that fails too, do the pass inline (see "Handling the findings").

## Handling the findings

The reviewer only reads — it returns what it found, and any edits to the spec are yours to make. It marks each finding decision or fix, but it never saw the conversation, so the sorting is yours. Sort each finding into one of three groups. First ask whether the user already settled it; if not, ask **is this a decision, or a fix?**

- **Already decided — note it, don't ask.** The reviewer never saw the conversation, so it will sometimes re-open something the user already settled. That might be the approach they chose before the draft was written, an answered Open Question, a `////` comment, or a reviewer suggestion they rejected earlier. Don't put it to them again. Make sure the spec states the decision plainly, since a finding like this usually means it was only implied, and note it in a line. Only this conversation counts, and only a clear answer: if you're unsure the user really settled it, it's a decision.
- **Decisions go to the user.** If resolving it means choosing between plausible alternatives — which of two designs was intended, whether scope has crept too far, which side of a UX tradeoff to take — a human has to make that call. These are the high-level items the review exists to surface. Bring them back as a short list: the category, the spot in the spec, what's at stake, and the options. Hold to critical/high here — the user just signed off, and interrupting that is earned only by something they'd genuinely want to decide before walking away; a pile of things to adjudicate trains them to skip the review. If nothing rises to that bar, say so in one line. Then let *them* choose what changes — reworking an approved design behind their back takes the decision out of their hands. When they pick something, revise the spec for just those points, and keep the Outcome bullets and the strategy in sync with whatever moved. When the user rejects a reviewer suggestion, add one line to Alternatives considered saying what was suggested and why the spec stays as it is, so builders and later reviews don't raise it again.
- **Fixes just get made.** If the problem has one obviously-correct resolution and no real tradeoff — a snippet that contradicts a stated decision, a missing null guard, two sections that disagree, an off-by-one in an example — there's nothing for the user to decide. Correct it directly in the spec (the spec file is the one you're allowed to edit), then note what you tidied in a line or two so nothing changes silently. These fixes make the spec match the design the user already approved; they don't re-open it. Forcing the user to adjudicate a mechanical correction wastes the very attention the review is meant to protect.

The line between a decision and a fix is simply whether more than one resolution is plausible. If a "fix" turns out to need a judgment call, it was never a fix — treat it as a decision and surface it. Either way, don't re-run the full review after every touch-up — it's a final gate, not a treadmill; a quick check that the flagged items are resolved is enough.

If subagents aren't available (e.g., Claude.ai), do the pass inline instead, right before asking for sign-off, since it can't overlap with the user's reading. Re-read the spec cold from disk, look through the six lenses under "For the reviewer", and apply the same split: surface the decisions, fix the mechanical defects. It's weaker — you can't truly un-know the conversation — but reading the written words fresh still catches more than skipping the pass.

### Mistakes to avoid

- **Bringing the user things that aren't decisions.** The review earns its keep by only surfacing genuine, critical/high design decisions — the calls a human actually needs to make. Mechanical defects with one right answer get fixed in the spec, not raised, and anything the user already settled in the conversation gets a one-line note, not a second question. Dumping fixable nits on someone who just signed off teaches them to ignore the review; if there's nothing to decide, say so in one line and stop.
- **Getting the decide/fix split backwards.** The two failure modes are opposite: making the user adjudicate a null-guard-grade fix, or quietly reworking the *design* under the cover of a "fix." The test is whether more than one resolution is plausible — if it is, it's a decision and belongs to the user.
