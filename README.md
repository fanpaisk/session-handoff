# session-handoff

**v0.1 — early, and honest about it.** Read [Validation status](#validation-status) before you rely on it.

An [Agent Skills](https://agentskills.io/specification) skill that moves the knowledge inside a long agent session into three durable files, so the session itself can be thrown away.

## The problem

A long agent session gets more expensive and less reliable with every turn. Compaction is lossy, and what it discards first is the valuable part: **why** a design was chosen, and which approaches already failed. The summary then keeps costing tokens on every later turn.

The thing you are afraid of losing — "this session knows everything about my project" — is usually already gone by then. Compaction ate it. You are paying full price for a degraded copy.

## What this does

It moves *knowledge about the project* out of the conversation and into files. What stays in the session is only the working state — the debug trail you are mid-way through, which is the one thing a session is genuinely irreplaceable for.

Three files, each with exactly one job:

| File | Holds | Auto-loaded? |
|---|---|---|
| `AGENTS.md` | The constitution: startup pointer, one-line positioning, hard constraints, build/test/acceptance commands, known traps | **Yes** |
| `docs/STATUS.md` | Status: deliverables and how each was verified, next step, open questions, known defects | No |
| `docs/DECISIONS.md` | Rationale: each decision, why, and what was rejected | No |

Every fact has exactly one home. The same fact in two files becomes two contradictory truths later.

**The load-bearing detail:** hosts auto-load `AGENTS.md` (DSH, ZCode, Claude Code, Codex) but not `docs/STATUS.md` or `docs/DECISIONS.md`. So `AGENTS.md` carries a mandatory pointer that makes a brand-new session read the other two. Get that pointer right and the next session needs only the word "continue".

## Install

A skill is a plain directory bundle: `SKILL.md` plus optional `references/`, `scripts/`, `evals/`. This one has no runtime dependencies and no scripts.

**Shared cross-tool root** — read by DSH and by other `SKILL.md`-compatible agents:

```bash
git clone https://github.com/fanpaisk/session-handoff ~/.agents/skills/session-handoff
```

**Per-tool roots**, if your tool does not scan the shared one:

| Tool | Path |
|---|---|
| DSH | `~/.agents/skills/` or `$DSH_HOME/skills/` |
| Claude Code | `~/.claude/skills/` or `<repo>/.claude/skills/` |
| Codex | `~/.codex/skills/` |
| Project-local (any) | `<project>/.agents/skills/` |

Some tools scan **one level only**: the bundle must be `<root>/<name>/SKILL.md`. A `SKILL.md` nested any deeper is not discovered.

## Use

After a feature passes acceptance, and before you open a new session:

```
Use session-handoff
```

It will:

1. Locate the project root, check whether the handoff files are under version control, and retire any competing status documents.
2. **Re-run** the build/test/acceptance commands — not recall them. A compacted session's memory of "tests passed" may refer to a checkout that no longer exists.
3. Write the three files, wrapping the `AGENTS.md` contribution in idempotent `<!-- SESSION_HANDOFF:START -->` / `<!-- SESSION_HANDOFF:END -->` markers, so re-running replaces rather than duplicates it.
4. Run a self-check, then emit a five-section handoff report.

Then open a fresh session and paste the opener it gives you — or, if the pointer is in place, just say "continue".

### Verifying the handoff actually worked

The only reliable acceptance test is a **cold read**: open a session with no history and ask it to restate what the project is, its hard constraints, its current status, and the next step. If it cannot, the files have a gap. **Fix the files — never explain it back in the old session.** Explaining it back is precisely the dependency this skill exists to remove.

## Validation status

**v0.1, with limited real-world use. Read this section before depending on it.**

| | Status |
|---|---|
| [Agent Skills spec](https://agentskills.io/specification) conformance | ✅ Verified mechanically — `name`, `description`, directory layout; frontmatter carries no tool-specific fields, so it ports between hosts |
| Real user handoff runs | **1**, on a 22k-file project, using an earlier revision of this skill |
| Evaluator runs (author, on a disposable copy of that project) | **1** |
| Cold-read acceptance test | **1 pass** — a zero-context agent reconstructed the project from the three files alone, and surfaced 23 gaps and contradictions while doing it |
| `evals/evals.json` | Written: 8 cases, 42 assertions — **never executed** |
| `evals` fixtures | **Not built.** The 5 fixtures are described in `evals.json` but do not exist, so the suite cannot run yet |
| Idempotent marker path | Exercised once |
| Pre-marker migration path | Exercised once, on a hand-built scenario. The rule it produced was added *because* that run failed |
| Credential-leak scenario | Exercised once — 0 leaks, on a project with a live secrets file in its root |

Two findings from those runs are why the skill is shaped the way it is:

- The rule **"re-run verification, do not recall it"** caught a recorded acceptance command that had silently gone stale: a later decision had changed the mechanism, and the old command no longer tested what it claimed. It had been sitting in a handoff marked "verified".
- A literal reading of the append rule produced **two contradictory copies** of the same constraints inside one `AGENTS.md`. The rule now distinguishes previous handoff output from user-written rules, and asks the user instead of guessing when it cannot tell.

## Known limitations

- **The skill body is written in Simplified Chinese.** The format is portable; the prose is not.
- **The "just say continue" claim depends on the host auto-loading project instruction files.** DSH, ZCode, Claude Code and Codex do. On a host that does not, the skill degrades gracefully — you paste the opener it emits — but the convenience is gone.
- **It cannot fix a project with no runnable acceptance criteria.** If "done" is not executable, the handoff will record that honestly, and nothing more.
- **Migrating a pre-existing handoff file needs judgment.** The skill asks the user when it cannot classify a section. It does not guess.
- **The evals are unexecuted** (see above). Treat them as a specification of intent, not as a passing suite.

## A note on how much handoff skills are worth

A generic handoff skill's value ceiling is *convenience*. Across one analysis of 673 agent skills, structural quality correlated near-zero with real usefulness (r ≈ 0.077). What actually helps is that the file records **your** project's specifics — the constraint that cost you a day, the approach you rejected and why. No skill can supply that content.

This skill's job is narrower and more defensible: make sure that content gets written down, and run one honest check that the result actually works.

## License

MIT — see [LICENSE](LICENSE).
