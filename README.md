# @atalanta/go-review-lenses

Three adversarial Go code-review prompts ("lenses") for an independent
reviewer in a `@swamp/software-factory` run — the Go analogue of
[`@atalanta/ts-review-lenses`](https://github.com/Atalanta/swamp-ts-review-lenses):
same three-pass structure, same `swamp`/findings contract, same
`IDIOM-`/`CORR-`/`ARCH-` id prefixes, but every lens is written for Go
specifically rather than adapted from the TypeScript prose.

| Lens | File | Obsession | Finding id prefix |
| --- | --- | --- | --- |
| A — Idiom | `references/lens-a-idiom.md` | Idiomatic with-the-grain Go, small interfaces at point of use, errors as values, enums via `iota`, no transliteration | `IDIOM-` |
| B — Correctness vs reality | `references/lens-b-correctness.md` | Boundary validation, the nil-interface trap, goroutine/slice-aliasing correctness, determinism, resource cleanup, grounded tests | `CORR-` |
| C — Architecture | `references/lens-c-architecture.md` | Package/dependency direction, the purity seam, change confinement, no speculative generality | `ARCH-` |

## Quick start

```bash
swamp extension pull @atalanta/go-review-lenses
# then invoke the bundled `go-review-lenses` skill
```

The skill documents how to wire the lenses into a factory's `plan-review`/
`code-review` stages and how to run each lens as an independent external
reviewer via
[`@atalanta/external-reviewer`](https://github.com/Atalanta/swamp-external-reviewer)'s
`external-review-findings` workflow (optional companion, not a hard
dependency) — or as a same-context `dispatch` fallback when no external
agent is configured.

## Running one lens by hand

A complete reviewer prompt is one lens file plus the swamp/findings contract,
concatenated at run time:

```bash
swamp workflow run @atalanta/external-reviewer/external-review-findings \
  --input factoryName=my-factory \
  --input workItem=ISSUE-42 \
  --input artifact=code-review \
  --input cwd=. \
  --input prompt="$(cat .claude/skills/go-review-lenses/references/lens-b-correctness.md
                    cat .claude/skills/go-review-lenses/references/contract-block.md)"
```

Run each lens as its own invocation (one per pass) so idiom, correctness,
and architecture findings stay distinct under their own id prefix — see the
bundled skill for the full drive loop, including how to read the parsed
findings back off the reviewer's `invocation` resource.

## Provenance

Sources: publicly available Go best practices — Ben Johnson's ["Standard
Package Layout"](https://medium.com/@benbjohnson/standard-package-layout-7cdbc8391fc1),
Peter Bourgon's ["Go best practices, six years
in"](https://peter.bourgon.org/go-best-practices-2016/), and Uber's public Go
Style Guide — plus accumulated wisdom contributed by this project's author,
checked against Go community idiom before folding it in. See
[`references/using-with-external-reviewer.md`](.claude/skills/go-review-lenses/references/using-with-external-reviewer.md)
for the full picture, including the couple of places an author instinct
toward clean/hexagonal architecture was softened to the idiomatic Go version
rather than folded in as-is.

## License

MIT. See `LICENSE.txt`.
