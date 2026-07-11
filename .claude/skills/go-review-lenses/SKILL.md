---
name: go-review-lenses
description: >-
  Wire and run three adversarial Go review lenses — idiom (idiomatic
  with-the-grain Go, small interfaces, errors as values), correctness-vs-reality
  (boundary validation, nil-interface trap, goroutine/slice-aliasing
  correctness, grounded tests), architecture (package boundaries, purity seam,
  dependency direction, change confinement — drawing on Ben Johnson's
  "Standard Package Layout" and Peter Bourgon's "Go best practices, six years
  in") — in a @swamp/software-factory run. Use when authoring or driving a
  factory's plan-review or code-review stage for a Go codebase and these
  lenses are present: at author time, set the review stages up for the three
  lenses; at drive time, run each lens as an independent external reviewer via
  @atalanta/external-reviewer's external-review-findings workflow and record
  its findings. Also covers the same-context dispatch fallback when no
  external agent is configured.
---

# Go Review Lenses

Three adversarial code-review prompts for Go, distilled from failure modes that
escape *green* test suites — including some caught in this repo's own history
(a test that duplicated production logic instead of exercising the real call
path; a test that reused shared state across cases and could have masked a
real regression) — and tempered against a personal Go architecture standard
(itself substantially derived from Uber's public Go Style Guide), keeping what
matches Go community consensus and explicitly rejecting the two pieces that
fought the language's actual grain. See
[references/using-with-external-reviewer.md](references/using-with-external-reviewer.md)'s
Provenance section for exactly what was kept, softened, or discarded and why.
Run them as **three separate passes** — one reviewer per lens — so idiom,
correctness, and architecture concerns don't dilute each other.

| Lens | File | Obsession | Finding id prefix |
| --- | --- | --- | --- |
| A — Idiom | `references/lens-a-idiom.md` | Idiomatic with-the-grain Go, small interfaces at point of use, errors as values, no transliteration from another language | `IDIOM-` |
| B — Correctness vs reality | `references/lens-b-correctness.md` | Boundary validation, the nil-interface trap, goroutine/slice-aliasing correctness, grounded tests | `CORR-` |
| C — Architecture | `references/lens-c-architecture.md` | Package/dependency direction, the purity seam, change confinement, no speculative generality | `ARCH-` |

A complete reviewer prompt is **one lens file + `references/contract-block.md`**
(the swamp/findings contract). The lens prose is transport-neutral; the contract
is what makes the output a recordable factory artifact. When two lenses conflict,
**correctness (B) outranks idiom (A)**.

Pick lenses by stage: `code-review` → all three; `plan-review` → C, optionally A
(idiom/correctness need real code to bite).

## When you are authoring a factory (set the review stages up for the lenses)

Trigger: you are filling a `@swamp/software-factory` definition's `plan-review`
or `code-review` stage and these lenses are available.

```
Author checklist:
- [ ] Review stages are mode: dispatch with a kind: findings artifact
- [ ] The findings artifact schema matches references/contract-block.md
      (id, severity critical|high|medium|low, description, category?, resolved?, resolutionNote?)
- [ ] code-review uses lenses A+B+C; plan-review uses C (+A)
- [ ] Decide the transport: external reviewer (default) or dispatch fallback
- [ ] Note in the stage description that findings come from N per-lens passes
      with prefixes IDIOM-/CORR-/ARCH-
- [ ] swamp model method run <factory> validate  → passes
```

Do **not** inline the lens prose into the factory YAML — keep it referenced from
these files so it stays versioned and editable in one place. The stage's job is
only to declare the findings artifact and its gates; the prompt is assembled at
drive time (below).

## When you are driving a review stage (run the lenses)

Trigger: a factory run has reached a `plan-review` or `code-review` stage whose
findings should come from these lenses.

Run **one external-reviewer invocation per lens**, then record the merged
findings. Use the Go module's repo root as `cwd` so the reviewer's
`swamp data query` reads the right datastore and `go.mod`/package tree.

```
Drive checklist (repeat the invoke+record for each chosen lens):
- [ ] record_dispatch on the factory stage (per the software-factory drive loop)
- [ ] Lens B (CORR-):   invoke → query invocation → record_artifact/resolve_findings
- [ ] Lens A (IDIOM-):  invoke → query invocation → record_artifact/resolve_findings
- [ ] Lens C (ARCH-):   invoke → query invocation → record_artifact/resolve_findings
- [ ] Re-check factory status; advance when findings-clear passes
```

**Step 1 — invoke the lens.** Concatenate the lens file and the contract into the
`prompt` input (two `cat`s — the lens prose then the contract; do **not** `cat`
`using-with-external-reviewer.md`, which is explanation, not contract):

```bash
swamp workflow run @atalanta/external-reviewer/external-review-findings \
  --input factoryName=my-factory \
  --input workItem=ISSUE-42 \
  --input artifact=code-review \
  --input cwd=. \
  --input prompt="$(cat .claude/skills/go-review-lenses/references/lens-b-correctness.md
                    cat .claude/skills/go-review-lenses/references/contract-block.md)"
```

**Step 2 — read the findings back.** `invokeAndParse` persists the parsed JSON on
the reviewer model's `invocation` resource, not as a step output:

```bash
swamp data query 'modelName == "external-reviewer" && specName == "invocation" \
  && attributes.tags.workItem == "ISSUE-42" \
  && attributes.tags.artifact == "code-review"' \
  --select '{"findings": attributes.parsedResponse, "ok": attributes.success}' --json
```

**Step 3 — record on the factory.** For the first recording of an artifact this
cycle use `record_artifact`; use `resolve_findings` only to mark specific
already-recorded findings as resolved on a later cycle. The `IDIOM-`/`CORR-`/
`ARCH-` prefixes keep the three passes' ids from colliding when the
`findings-clear` gate evaluates them.

**Step 4 — repeat** for the remaining lenses, then advance the factory once
`findings-clear` is satisfied. Substitute `--input artifact=plan-review` for a
plan-review stage.

If the reviewer returns `ok: false` or empty `findings`, do not re-dispatch
blindly — read the invocation's `outputPreview` / transcript for why (auth,
provider, malformed JSON) per the software-factory dispatch guard.

## Fallback: same-context dispatch (no external agent)

If no external reviewer is configured, run the review stages as vanilla
`mode: dispatch` with one subagent per lens: each subagent's `systemPrompt` is
the lens file + `references/contract-block.md`, and it reviews the diff from its
own context instead of via `swamp data query`. Merge into one `kind: findings`
artifact with the same id prefixes. This shares the implementer's model family,
so it is review but not *independent* review — prefer the external path when both
are available.

## References

- [references/lens-a-idiom.md](references/lens-a-idiom.md) — Lens A prompt
- [references/lens-b-correctness.md](references/lens-b-correctness.md) — Lens B prompt
- [references/lens-c-architecture.md](references/lens-c-architecture.md) — Lens C prompt
- [references/contract-block.md](references/contract-block.md) — paste-clean swamp/findings contract appended to a lens at run time
- [references/using-with-external-reviewer.md](references/using-with-external-reviewer.md) — how the lens + contract become a prompt, plus provenance (explanation, not for `cat`)

All three lenses are Go-specific by design (unlike the TS lens set's B/C, which
are largely language-portable, these name Go's actual pitfalls — nil interfaces,
slice aliasing, goroutine leaks, package-level dependency direction). For a
non-Go Go-adjacent language, treat B as mostly portable and adapt A/C.
